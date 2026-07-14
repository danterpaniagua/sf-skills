# Skill: AWS / SQS Operations Triage

Diagnose and resolve AWS SQS operational issues in platforms-service and concentrador-service.

Run this skill when orders are stuck, consumers are down, dead letters are accumulating, or any SQS-related error appears in `logerrors`.

---

## Architecture reference

```
PedidosYa / Rappi / UberEats / MercadoPago / PediGrido
        ↓  webhook
  platforms-service
        ↓  pushNewToQueue
  {branchId}_PlatformMessages.fifo  (per-branch, us-east-2)
        ↓  pollFromQueue (parent consumer)
            + up to 3 child consumers (auto-scale at 200 / 400 / 600 depth)
        ↓  setNews → news state transition
  BranchMessages.fifo → SmartFran agent (POS terminal)

  On failure → DeadLetter.fifo / MainDeadLetter.fifo
        ↓  concentrador-service deadLetterSave() on startup
  smartfran.deadletters  (MongoDB)
```

**Queue URLs (testing):**
- Producer: `https://sqs.us-east-2.amazonaws.com/382381053403/{branchId}_TST_PlatformMessages.fifo`
- Consumer: `https://sqs.us-east-2.amazonaws.com/382381053403/TST_BranchMessages.fifo`
- Dead letter: `https://sqs.us-east-2.amazonaws.com/382381053403/DeadLetter.fifo`
- Main dead letter: `https://sqs.us-east-2.amazonaws.com/382381053403/MainDeadLetter.fifo`

**Key MongoDB collections:** `logerrors` · `deadletters` · `news` · `configs`

**Key flag:** `configs.saveDeadLetterbol` — if `false`, concentrador skips dead letter recovery silently.

---

## Failure catalog

### F-001 · SQS consumer down (platforms-service)

**Symptom:** Orders received by platforms-service but never arriving at POS terminal. No activity in `logerrors` for `setNews_processing`.

**Log signals:**
```
[INFRA/ERROR] sqs_consumer failed event=error consumer=parent
[INFRA/ERROR] sqs_consumer failed event=processing_error consumer=parent
[INFRA/WARN]  sqs_consumer failed event=timeout_error consumer=parent
```

**Diagnostic queries:**
```js
// 1. Recent consumer errors
db.logerrors.find({
  service: 'platforms-service',
  message: { $regex: 'sqs_consumer' },
  createdAt: { $gte: new Date(Date.now() - 60*60*1000) }
}).sort({ createdAt: -1 }).limit(20)

// 2. Is setNews processing anything?
db.logerrors.find({
  service: 'platforms-service',
  message: { $regex: 'setNews_processing' },
  createdAt: { $gte: new Date(Date.now() - 30*60*1000) }
}).sort({ createdAt: -1 }).limit(10)

// 3. Check error_code and retryable flag
db.logerrors.find({
  service: 'platforms-service',
  'error.customDataAfter.error_code': { $exists: true }
}, { 'error.customDataAfter.error_code': 1, 'error.customDataAfter.retryable': 1 }).limit(10)
```

**Remediation:**
- `retryable: true` → transient AWS issue; consumer will restart automatically
- `error_code: CredentialsProviderError` → ECS task role misconfigured; check IAM permissions
- `error_code: QueueDoesNotExist` → queue was deleted; check branch provisioning
- Consumer does not auto-restart after `event=error` — ECS task restart may be needed

---

### F-002 · Order stuck in queue / setNews failure (platforms-service)

**Symptom:** Order received, pushed to queue, but news state not updating in MongoDB.

**Log signals:**
```
[ORDER/ERROR] setNews_processing failed consumer=parent
[ORDER/ERROR] setNews_processing failed consumer=child
[INFRA/ERROR] batch_processing failed consumer=parent  ← SendGrid alert also fired
```

**Diagnostic queries:**
```js
// 1. Which orders are failing?
db.logerrors.find({
  service: 'platforms-service',
  message: { $regex: 'setNews_processing' }
}, { 'error.message': 1, 'error.customDataAfter': 1, createdAt: 1 })
  .sort({ createdAt: -1 }).limit(20)

// 2. Find the stuck news document
db.news.find({ _id: ObjectId('<order_id_from_log>') })

// 3. Check news stuck in a transition state
db.news.find({
  typeId: { $in: [1, 2, 3] },
  updatedAt: { $lt: new Date(Date.now() - 30*60*1000) }
}).limit(20)

// 4. Dead letters from this order
db.deadletters.find({ 'bodyParsed.id': '<order_id>' })
```

**Remediation:**
- If order is in `deadletters` → use `pushAutoReplyToQueue` sequence (typeId: 10 → 11 → 5 → 3 → 14) to force state resolution
- If news is stuck with `typeId: 1` (received) → platform rejected after timeout; manual rejection may be needed
- `batch_processing failed` means the entire batch was unprocessable — SendGrid alert was sent to `smartpedidos@smartfran.com`

---

### F-003 · Dead letter accumulation (concentrador-service)

**Symptom:** Orders missing from POS but no error in platforms-service. `deadletters` collection growing.

**Log signals:**
```
[INFRA/ERROR] sqs_dead_letter_save failed queue=DeadLetter
[INFRA/ERROR] sqs_dead_letter_save failed queue=MainDeadLetter
[INFRA/ERROR] sqs_delete_message failed
[INFRA/ERROR] json_parse failed
```

**Diagnostic queries:**
```js
// 1. Is dead letter recovery enabled?
db.configs.findOne({}, { saveDeadLetterbol: 1 })
// → if false: dead letters are being silently skipped

// 2. How many dead letters accumulated?
db.deadletters.aggregate([
  { $group: { _id: '$type', count: { $sum: 1 }, last: { $max: '$fechaGuardado' } } }
])

// 3. Recent dead letter save failures
db.logerrors.find({
  service: 'concentrador-service',
  message: { $regex: 'dead_letter' },
  createdAt: { $gte: new Date(Date.now() - 2*60*60*1000) }
}).sort({ createdAt: -1 }).limit(20)

// 4. Inspect dead letter messages
db.deadletters.find({ type: 'order' }).sort({ fechaGuardado: -1 }).limit(10)

// 5. JSON parse failures — malformed messages
db.logerrors.find({
  service: 'concentrador-service',
  message: '[INFRA/ERROR] json_parse failed'
}).sort({ createdAt: -1 }).limit(10)
```

**Remediation:**
- `saveDeadLetterbol: false` → set to `true` and restart concentrador; `deadLetterSave()` runs on startup
- `sqs_dead_letter_save failed` → MongoDB write failed; check Atlas connectivity
- `sqs_delete_message failed` → message will be retried on next pull; check if DB write succeeded first (avoid duplicate saves)
- `json_parse failed` → malformed message body; inspect `deadletters` for raw `body` field
- Dead letters persist for 3 days (`MessageRetentionPeriod: 259200`); act within that window

---

### F-004 · Queue push failure — order not reaching branch (platforms-service)

**Symptom:** Order confirmed by platform but branch POS never receives it.

**Log signals:**
```
[INFRA/ERROR] pushNewToQueue failed
[INFRA/ERROR] pushAutoReplyToQueue failed
```

**Diagnostic queries:**
```js
// 1. Which branches have push failures?
db.logerrors.find({
  service: 'platforms-service',
  message: { $regex: 'pushNewToQueue|pushAutoReplyToQueue' }
}, { 'error.customDataAfter.branch_id': 1, 'error.customDataAfter.order_id': 1, 'error.message': 1, createdAt: 1 })
  .sort({ createdAt: -1 }).limit(20)

// 2. Does the branch queue exist?
// → check AWS console for queue {branchId}_PRD_PlatformMessages.fifo

// 3. Has the order been received?
db.orders.findOne({ _id: ObjectId('<order_id>') })
db.news.findOne({ 'order._id': ObjectId('<order_id>') })
```

**Remediation:**
- `QueueDoesNotExist` → branch queue was never created or was deleted; re-trigger branch provisioning (`createQueues(branchId)`)
- `AccessDeniedException` → IAM role missing `sqs:SendMessage` on target queue
- If order is in MongoDB but not in queue → manually push via `pushAutoReplyToQueue` or recreate the news document

---

### F-005 · Consumer queue depth buildup (platforms-service)

**Symptom:** Orders delayed, high latency, secondary consumers should be active.

**Context:** platforms-service auto-scales consumers based on `ApproximateNumberOfMessages`:
- Depth > 200 → spawns 2nd consumer
- Depth > 400 → spawns 3rd consumer
- Depth > 600 → spawns 4th consumer

**Diagnostic queries:**
```js
// 1. Volume of setNews processing in last hour
db.logerrors.aggregate([
  { $match: { service: 'platforms-service', message: { $regex: 'setNews_processing' },
    createdAt: { $gte: new Date(Date.now() - 60*60*1000) } } },
  { $group: { _id: { $dateToString: { format: '%H:%M', date: '$createdAt' } }, count: { $sum: 1 } } },
  { $sort: { _id: 1 } }
])

// 2. PERF — is the consumer slow?
db.logerrors.find({
  service: 'platforms-service',
  category: 'PERF',
  createdAt: { $gte: new Date(Date.now() - 30*60*1000) }
}, { message: 1, 'error.customDataAfter.duration_ms': 1 })
  .sort({ 'error.customDataAfter.duration_ms': -1 }).limit(10)

// 3. getQueueAttributes failures (can't read depth)
db.logerrors.find({
  service: 'platforms-service',
  message: '[INFRA/WARN] getQueueAttributes failed'
}).sort({ createdAt: -1 }).limit(5)
```

**Remediation:**
- High depth + slow PERF → external platform API latency; check `[PERF/INFO]` duration_ms per platform
- `getQueueAttributes failed` → auto-scaling is blind; child consumers won't spawn; manual ECS scale-out may be needed
- If depth is sustained > 600 → consider increasing ECS task count directly

---

## Standard first-response queries

Run these for any SQS issue before diving into specific failure patterns:

```js
// A. Error volume last 2 hours by message type
db.logerrors.aggregate([
  { $match: { category: 'INFRA', createdAt: { $gte: new Date(Date.now() - 2*60*60*1000) } } },
  { $group: { _id: '$message', count: { $sum: 1 } } },
  { $sort: { count: -1 } }, { $limit: 15 }
])

// B. Is concentrador dead letter flag on?
db.configs.findOne({}, { saveDeadLetterbol: 1 })

// C. Dead letter count by type
db.deadletters.aggregate([
  { $group: { _id: '$type', count: { $sum: 1 }, oldest: { $min: '$fechaGuardado' } } }
])

// D. Orders stuck in early states
db.news.countDocuments({ typeId: { $lte: 3 }, updatedAt: { $lt: new Date(Date.now() - 30*60*1000) } })
```

---

## IAM Credential Leak — Impact Verification

When a credential is found exposed (in logs, a response body, or source) and someone claims "no real impact" — verify empirically, don't accept the claim or restate the policy name alone. Confirmed workflow (2026-07-13, `userSQS` leak):

```bash
# 1. What's actually attached — managed policies, inline policies, group membership
aws iam list-attached-user-policies --user-name <user> --output table
aws iam list-user-policies --user-name <user> --output table
aws iam list-groups-for-user --user-name <user> --output table

# 2. Read every attached managed policy's actual document — Resource scoping is the whole question
aws iam get-policy --policy-arn <policy-arn> --query "Policy.DefaultVersionId" --output text
aws iam get-policy-version --policy-arn <policy-arn> --version-id <DefaultVersionId> --query "PolicyVersion.Document" --output json

# 3. Empirical proof, not just the document — IAM Policy Simulator against representative
#    actions (include destructive ones: PurgeQueue, DeleteObject, etc., not just read actions)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<account>:user/<user> \
  --action-names <action1> <action2> <action3> \
  --resource-arns "*" \
  --output table

# 4. Key status and freshness — a key can be Active with an old CreateDate; check both,
#    and pull a *fresh* last-used timestamp rather than reusing an earlier finding's date
aws iam list-access-keys --user-name <user> --output table
aws iam get-access-key-last-used --access-key-id <AccessKeyId> --output json
```

**What this answers, concretely:** whether the credential is scoped to specific resources or wildcarded (`Resource: "*"`), whether the specific leaked key (not just "a key on this user") is Active or already rotated, and whether it's actually been used recently — all three are required before agreeing or disagreeing with an impact claim. A policy *name* like `AmazonSQSFullAccess` is not evidence on its own — always pull the document and run the simulator.

---

## Tracing When a Finding Was Introduced

When multiple instances of the same defect pattern exist across files/services, check whether they came from one change or were introduced independently — it changes the fix (revert one commit vs. structural detection):

```bash
# Run from the affected repo's local clone (e.g. smartfran/sp-logs/repo/<service>/)
git log --oneline -- <file>
git blame -L <start>,<end> --date=short -- <file>
```

Compare commit hashes and dates across every instance. Per the no-names rule below, cite commit id + date only in anything written to `events/` or a ticket — never the author name (fine to say out loud in conversation, not in the written record).
