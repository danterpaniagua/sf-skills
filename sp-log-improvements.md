# Skill: Log Improvements

Apply the SmartPedidos logging standard to a service codebase. Run this skill when starting a new log improvement cycle on any service.

## Reference files

- Standards: `docs/logging-standards.md`
- Completed example: `docs/jira-story-log-improvements.md` (platforms-service)
- Prior findings: `docs/<service>-findings.md`

## Workflow

### Step 1 — Scan

Search the target service for every antipattern below. Collect file + line evidence before writing anything.

| Check | What to look for |
|---|---|
| SUB-000 · Credentials in logs | `Authorization`, `rappi-signature`, `x-api-key`, `cookie`, `headersConfig`, `req.headers` passed raw to any log call |
| SUB-001 · Wrong platform names | Log messages that name a platform different from the file/class they are in |
| SUB-002 · Double logging | Inner `catch` that calls `Log.saveError` and then `throw error` — same error logged twice |
| SUB-003 · Wrong log levels | Token retries, transient failures, or expected branch states logged as `ERROR` instead of `WARN` |
| SUB-004 · Spanish messages | Any non-English characters or Spanish words in log message strings |
| SUB-005 · Missing level/category | Log calls with no `level` or `category` field |
| SUB-006 · Missing tracing fields | `ORDER` or `INTEGRATION` log calls missing `branch_id`, `platform_id`, or `order_id` when in scope |
| SUB-007 · No PERF logging | HTTP calls to external platforms with no `[PERF/INFO]` timing entry. If the service has `utils/httpClient.js`, check that all outbound calls use it instead of raw `axios` — the interceptor already handles PERF logging automatically via `_perf` config metadata |
| SUB-008 · console.log in production | Any `console.log(error)` preceding or replacing a `Log.save` call |
| SUB-009 · Missing service field | `Log` model does not inject `process.env.SERVICE_NAME` automatically |
| SUB-010 · Log responsibility concentration | Files where log calls are disproportionately concentrated — assess whether logging should be withdrawn from the file and delegated to a base class, middleware, or shared helper. Established pattern: `utils/httpClient.js` (Axios interceptor) already withdrew PERF log responsibility from controllers. Remaining gap: error logging still lives inline in each controller |

### Step 2 — Write findings

Create or append to `docs/<service>-findings.md`. Never overwrite existing findings — append a new dated section.

Each finding must include: severity, file + line, current code (bad), correct code (good).

For SUB-010, include a feasibility assessment with:
- File name and total log call count vs. service total (e.g. `branch.js — 88 / 131 calls = 67%`)
- Root cause: mixed concerns (controller + platform orchestration + cron logic), missing base class, no event layer, etc.
- Recommendation: extract to base class, middleware, event emitter, or dedicated logging wrapper — with a concrete example
- Effort estimate: Low / Medium / High

Ask before writing: "Do you want me to write the findings file for this run?"

### Step 3 — Implement

Apply fixes in this priority order: SUB-000 (security, always first) → SUB-010 (log responsibility — must precede SUB-007 and SUB-008, as its outcome changes how those are implemented) → SUB-002 → SUB-001 → SUB-003 → SUB-005 → SUB-009 → SUB-004 → SUB-006 → SUB-007 → SUB-008.

Rules:
- Edit existing files only — never rewrite whole files.
- Log changes must never affect control flow, return values, error propagation, or business logic.
- `Log.saveError` must remain as a backwards-compatible delegate to `Log.save('ERROR', ...)`.
- Null-safe fallback chain on all tracing fields: `primary ?? secondary ?? 'unknown'`. Never leave tracing fields absent.
- For SUB-007: preferred implementation is `utils/httpClient.js` (Axios interceptor pattern from concentrador-service). Create it if it does not exist; migrate raw `axios` calls to use it with `_perf` config metadata. Do not add inline timing to individual call sites.

### Step 4 — Validate

For each sub-task, confirm with a grep or read that the fix is applied. Report pass/fail per item.

Validation checklist:
- No `Authorization`, `rappi-signature`, or raw `req.headers` in any `Log.save` call
- `sanitizeHeaders()` exists in `log.js` and is used at all inbound header log points
- No Spanish characters in any log message string
- Every `Log.save` call has `level` and `category`
- Every `ORDER`/`INTEGRATION` log call has `branch_id`, `platform_id`, `order_id` with `'unknown'` fallback
- No `console.log(error)` adjacent to a `Log.save` call
- `log.js` reads `process.env.SERVICE_NAME` and injects it automatically
- Double-logging pattern eliminated in all token/retry flows
- `[PERF/INFO]` entries present on all external HTTP calls

### Step 5 — Generate Jira story

Create `docs/jira/<date>_<service>-log-improvements/ticket.md` following the structure in `docs/jira-story-log-improvements.md`. Include: description, acceptance criteria (one per sub-task), sub-task list with files and line numbers, final status.

If SUB-010 findings were identified, add a dedicated section to the ticket: **"Log responsibility — feasibility"** with the assessment and recommendation. Mark it as a separate follow-up story if the effort is Medium or High.

Ask before writing: "Do you want me to generate the Jira story for this run?"

## Acceptance criteria (applies to every service)

- [ ] No bearer tokens, HMAC signatures, or raw header objects in `logerrors`
- [ ] `sanitizeHeaders()` centralised in `log.js` and applied at all inbound header log points
- [ ] All log messages in English, no accented characters
- [ ] Every `Log.save` call includes `level` and `category`
- [ ] HTTP response times logged as `[PERF/INFO]` with `duration_ms` and `status`
- [ ] No `console.log(error)` in production code adjacent to a log call
- [ ] Double logging eliminated — each failure produces exactly one `logerrors` document
- [ ] Token retries logged as `WARN`; only unrecoverable failure as `ERROR`
- [ ] Platform names correct in all log messages
- [ ] All `ORDER`/`INTEGRATION` logs include `branch_id`, `platform_id`, `order_id` when in scope
- [ ] Every `logerrors` entry includes `service` field from `SERVICE_NAME` env var

## Message format standard

```
[CATEGORY/LEVEL] <action> <entity> key=value key=value
```

Examples:
```
[INTEGRATION/ERROR] token_fetch failed platform=MercadoPago user_id=123 attempt=2/3
[ORDER/ERROR] manageNewType failed strategy=DeliveryStrategy order_id=abc branch_id=456
[INFRA/ERROR] createConsumerSQS failed queue=sqs-customer-setNews
[PERF/INFO] http_response platform=PedidosYa action=receiveOrder duration_ms=342 status=200
```
