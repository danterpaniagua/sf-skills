# Skill: SRP Refactor

Analyse a target file or service for Single Responsibility Principle violations, produce a prioritised extraction plan, and generate a Jira story.

Applies to both SmartPedidos services: **concentrador-service** and **platforms-service**.

Run this skill when a file exceeds ~500 lines, mixes more than two distinct responsibilities, or when a log-improvements cycle surfaces a SUB-010 finding.

## Reference

| Service | SRP state | Key files |
|---|---|---|
| **concentrador-service** | Per-platform providers not yet extracted — all platform logic lives in `controllers/branch.js` (5,571 lines, 8 clusters) | `docs/concentrador-findings.md` IMP-009 |
| **platforms-service** | Per-platform providers already extracted (`api/src/platforms/management/platform/<name>.js`) with `Platform` base class — SRP work focuses on within-provider concerns and observability separation | `docs/platform-findings.md` |

Established extraction patterns:
- `api/src/utils/httpClient.js` (concentrador) — PERF log responsibility withdrawn from controllers via Axios interceptor
- `api/src/platforms/management/platform/` (platforms-service) — per-platform provider pattern; concentrador extractions should mirror this structure

## Workflow

### Step 1 — Map responsibilities

Read the target file. Group every function, method, and constant into responsibility clusters. A cluster is a set of symbols that share a single cohesive concern.

For each cluster produce:

| Field | Content |
|---|---|
| Name | Short label, e.g. `TokenManager`, `PlatformStatusSync`, `BranchCronScheduler` |
| Symbols | Function/method names that belong to it |
| Line range | Approximate span in the file |
| Line count | Total lines owned by this cluster |
| External deps | Imports / models / env vars exclusively used by this cluster |
| Coupling | Other clusters this one calls directly — high coupling = higher extraction risk |

Threshold: flag any cluster with >150 lines or any file with >3 clusters as an SRP violation.

### Step 2 — Score each extraction

For each cluster, assign:

| Score | Criteria |
|---|---|
| **Effort** Low / Medium / High | Low: pure functions, no shared state. Medium: shared module-level variables. High: deeply entangled with other clusters or exports used by many callers. |
| **Risk** Low / Medium / High | Low: no side effects on control flow. Medium: shared mutable state or circular deps. High: changes request/response pipeline or SQS consumer path. |
| **Value** Low / Medium / High | High: cluster has its own test surface, is reused elsewhere, or is the subject of frequent bugs. |

### Step 3 — Write findings

Append to `docs/<service>-findings.md` under a new dated section.

Each finding must include:
- Cluster name, line count, % of file
- Symbols list
- Effort / Risk / Value scores
- Recommended target file path (e.g. `api/src/services/tokenManager.js`)
- One concrete example: current location vs. proposed extraction

Ask before writing: "Do you want me to write the SRP findings for this file?"

### Step 4 — Implement

Apply extractions in this order: highest Value + lowest Risk first.

Rules:
- Never break exports — if the original file exports a symbol, re-export it from the original file after extraction to avoid breaking callers.
- Never change function signatures, return values, or error propagation.
- Move shared module-level state (e.g. token variables) into the new module's own scope.
- One extraction per commit.
- Do not extract and refactor in the same commit — move first, refactor separately.

### Step 5 — Validate

For each extracted cluster:
- Grep original file — confirm symbols no longer defined there (only re-exported if needed)
- Grep new file — confirm all symbols present
- Grep callers — confirm no broken imports
- Run service if possible to verify startup

### Step 6 — Generate Jira story

Create `docs/jira/<date>_<service>-srp-refactor/ticket.md`.

Include:
- Description: current state (line count, cluster count, coupling issues)
- One sub-task per cluster extraction, ordered by Value desc / Risk asc
- Acceptance criteria: each extracted file has a single responsibility, original file shrinks below 500 lines, no broken exports
- Follow-up marker if any High-risk extraction is deferred

Ask before writing: "Do you want me to generate the Jira story for this refactor?"

## Responsibility cluster taxonomy

Common clusters found in SmartPedidos controllers — use these labels for consistency:

| Label | What it owns |
|---|---|
| `TokenManager` | Login, token refresh, token cache, credential encryption/decryption |
| `PlatformStatusSync` | Sending open/closed state to each platform API |
| `BranchOpenClose` | Close/open branch across all platforms after a business event |
| `CronScheduler` | Cron definitions, schedule setup, timezone helpers |
| `OrderRejection` | Reject flows, dead-letter recovery, canje integration |
| `HttpController` | Express request handlers (req/res), input validation, response shaping |
| `PlatformOfflineCheck` | Polling each platform to detect offline POS state |
| `PerfInterceptor` | HTTP timing — belongs in `utils/httpClient.js`, never in a controller |

## Extraction priority (both services)

The priority order is identical for both services. Current state differs — check the table below before starting.

**1. Per-platform providers (highest priority)**

Each delivery platform must own its full lifecycle in a dedicated file. Extract all functions that communicate exclusively with one platform before addressing any other cluster.

Target structure:
```
api/src/providers/
  rappi.js          ← login, token, offline check, status sync, open/close
  pedidosYa.js      ← login, token, offline check, status sync, open/close
  uberEats.js       ← login, token, offline check, status sync, open/close
  mercadoPago.js    ← login, token, offline check, status sync, open/close
  pediGrido.js      ← login, token, offline check, status sync, open/close
```

**2. Branch provider**

Branch-level orchestration that coordinates across platforms (open/close all, offline check all, status sync all). No platform-specific HTTP calls:
```
api/src/providers/branch.js
```

**3. Remaining clusters**

Once providers are clean, whatever remains in the original controller should be `HttpController` only — request handlers, input validation, response shaping.

**4. Observability separation**

See section below. Apply after providers are clean so the event boundaries are stable before wiring up subscribers.

---

### Current state per service

| Priority | concentrador-service | platforms-service |
|---|---|---|
| 1 · Per-platform providers | ❌ Not extracted — all in `controllers/branch.js` | ✅ Done — `api/src/platforms/management/platform/<name>.js` |
| 2 · Branch provider | ❌ Not extracted | ✅ Done — `Platform` base class coordinates lifecycle |
| 3 · Remaining clusters | ❌ `branch.js` still has CronScheduler, TokenManager, OrderRejection mixed in | ⚠️ Partial — review individual platform files for within-provider SRP violations (threshold: >300 lines or >2 concerns per file) |
| 4 · Observability separation | ⚠️ PERF done via `httpClient.js` — error logging inline | ❌ Not started |

---

## Observability separation (SRE ownership boundary)

Target: log calls live in SRE-owned files; business logic files have zero `Log` imports. Eliminates merge conflicts between SRE and dev work.

**Pattern: event emitter.** Devs emit domain facts; SRE subscribes and decides level/fields.

```
// providers/rappi.js — dev owns
bus.emit('rappi.token.failed', { attempt, error, branch_id });

// observability/rappi.js — SRE owns
bus.on('rappi.token.failed', data =>
  Log.save('ERROR', data.error, { category: 'INTEGRATION', message: '[INTEGRATION/ERROR] rappi_token_failed', ...data })
);
```

`httpClient.js` is already this pattern for PERF — callers pass `_perf` metadata, never import `Log`.

**Constraint:** every catch block must throw or emit — never silently swallow. Enforce via code review.

Include observability separation as a dedicated sub-task in the Jira story.

