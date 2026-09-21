# Skill: App Service Performance Report

Builds one linked HTML report (Altair/Vega-Lite) correlating per-endpoint HTTP response time, request volume, response codes, and App Service Plan CPU/RAM against a shared, zoomable timeline with incident markers — for any of the 8 production SmartFran Cloud App Services.

Originated in `cloud/events/20260913_pos-net-http-timeout/` (Sales-PRO OOM investigation); generalized here for reuse. The report script itself lives at `cloud/scripts/appservice_perf_report.py` — this skill documents the 3-step workflow and the per-app values it needs.

## Scope

- Any of the 8 production App Services: Admin, Business, Catalog, Orders, Person, Platform, Pos, Sales.
- Correlating HTTP-level symptoms (slow/failing endpoints) against App Service Plan resource pressure (CPU/RAM) on one shared, interactive timeline.
- Not for DEV — the per-app `PROD-{App}-AppServiceHTTPLogs` Graylog streams (below) are PROD-only; see `cloud/docs/graylog-log-fields.md` for the DEV shared-stream layout if that's ever needed.

## Known Infrastructure

| Value | |
|---|---|
| Subscription | SmartIT Cloud (`85c76dea-3304-4310-8656-bf21b28e4f4b`) |
| Resource group (all 8 apps + plans) | `SmartFran.Cloud.PRO` |
| Graylog base URL | `https://sfcloud-monitoreo.smartfran.com/graylog/api` |

**Per-app resource names and Graylog stream IDs** (App Service / Plan follow `SmartFran-Cloud-{App}-PRO` / `SmartFran-Cloud-{App}-Plan-PRO`; stream IDs confirmed live 2026-08-26 in `cloud/events/20260826_appservicehttplogs-stream-split/investigation.md`):

| App | App Service | App Service Plan | `PROD-{App}-AppServiceHTTPLogs` stream id |
|---|---|---|---|
| Admin | `SmartFran-Cloud-Admin-PRO` | `SmartFran-Cloud-Admin-Plan-PRO` | `6a8edbc94814bcaed323ede1` |
| Business | `SmartFran-Cloud-Business-PRO` | `SmartFran-Cloud-Business-Plan-PRO` | `6a8edd554814bcaed323f891` |
| Catalog | `SmartFran-Cloud-Catalog-PRO` | `SmartFran-Cloud-Catalog-Plan-PRO` | `6a8edd584814bcaed323f8c8` |
| Orders | `SmartFran-Cloud-Orders-PRO` | `SmartFran-Cloud-Orders-Plan-PRO` | `6a8edd5c4814bcaed323f903` |
| Person | `SmartFran-Cloud-Person-PRO` | `SmartFran-Cloud-Person-Plan-PRO` | `6a8edd5f4814bcaed323f934` |
| Platform | `SmartFran-Cloud-Platform-PRO` | `SmartFran-Cloud-Platform-Plan-PRO` | `6a8edd624814bcaed323f968` |
| Pos | `SmartFran-Cloud-Pos-PRO` | `SmartFran-Cloud-Pos-Plan-PRO` | `6a8edd664814bcaed323f9a8` |
| Sales | `SmartFran-Cloud-Sales-PRO` | `SmartFran-Cloud-Sales-Plan-PRO` | `6a8edd694814bcaed323f9d6` |

**Pos caveat:** whether Pos actually emits `AppServiceHTTPLogs` at all was never confirmed (`cloud/docs/graylog-log-fields.md`) — if step 1 below returns ~0 rows for Pos, that's a known open question, not necessarily a bug in this workflow.

**Function Apps are out of this skill's scope** — `SFC-CodeGenerator-PRO`, `SFC-ScheduledJobs-PRO`, `SFC-PosWasmRelay-PRO`, `SFC-TicketProcess-PRO` don't emit `AppServiceHTTPLogs` (different category, `FunctionAppLogs`, different raw shape — this script's parser doesn't handle it). Two separate ways to get their traffic, both confirmed working (`cloud/events/20260915_gitin2042-waf-cost-estimate/`, `cloud/events/20260915_functionapp-onboarding/`):
- **Aggregate execution count, any window, no Graylog needed:** `az monitor metrics list --resource ".../Microsoft.Web/sites/<Function App>" --metric FunctionExecutionCount --aggregation Total --interval PT1H --start-time <ISO> --end-time <ISO>` — one number per app per hour, doesn't distinguish which function within the app fired.
- **Per-function breakdown, Graylog, only from 2026-09-15 onward (field promotion isn't retroactive):** each app has its own dedicated stream (`category:"FunctionAppLogs" AND name:"<APP-UPPERCASE>"` — `name` indexes uppercase, mixed-case returns 0 silently). Filter on `FunctionName` (+ `FunctionEventName:"FunctionCompleted"` for one row per real invocation, not one per log line) to separate e.g. an HTTP-triggered endpoint from timer-triggered siblings in the same app. Full field reference: `cloud/docs/graylog-log-fields.md` → `FunctionAppLogs` section.

## Workflow — 3 steps, one report

All commands are copy-paste blocks — the user runs them and pastes results back (or confirms completion), per this project's convention. Requires `$GRAYLOG_API_KEY` (never a literal token) and an authenticated `az` session.

### Step 1 — Graylog raw export (`CsUriStem`/`TimeTaken`/`ScStatus`/`Referer`)

Pulls raw, unparsed `properties` JSON — bypasses Graylog's field-promotion pipeline entirely (unreliable for historical data on some branches, see `cloud/docs/graylog-log-fields.md` GITIN-2034), so this always works regardless of promotion status.

```bash
STREAM_ID="<stream id from the table above>"
OUT_DIR="<event folder>/data"
START="<ISO timestamp, e.g. 2026-09-11T18:20:00.000Z>"
END=$(date -u '+%Y-%m-%dT%H:%M:%S.000Z')

curl -s -u "${GRAYLOG_API_KEY}:token" \
  --data-urlencode 'query=*' \
  --data-urlencode "filter=streams:${STREAM_ID}" \
  --data-urlencode 'fields=timestamp,properties' \
  --data-urlencode "from=${START}" \
  --data-urlencode "to=${END}" \
  -G "https://sfcloud-monitoreo.smartfran.com/graylog/api/search/universal/absolute/export" \
  -H 'Accept: text/csv' \
  -o "${OUT_DIR}/httplogs_raw_export.csv"

wc -l "${OUT_DIR}/httplogs_raw_export.csv"
```

### Step 2 — App Service Plan CPU/RAM (true avg/min/max, not derived)

Request all three aggregations explicitly — a plain pull only returns Average, which hides real bursts (confirmed case: CPU averaged 15% but peaked at 53% within a single 5-minute window). `PT10M` is **not** a valid Azure Monitor granularity (`PT1M/PT5M/PT15M/PT30M/PT1H/PT6H/PT12H/P1D` only) — use `PT5M`, which divides evenly into the script's 10-minute buckets.

```bash
SUB="85c76dea-3304-4310-8656-bf21b28e4f4b"
RG="SmartFran.Cloud.PRO"
PLAN="<App Service Plan name from the table above>"
OUT_DIR="<event folder>/data"
START="<same ISO timestamp as step 1>"
END=$(date -u '+%Y-%m-%dT%H:%M:%SZ')

az monitor metrics list \
  --resource "/subscriptions/${SUB}/resourceGroups/${RG}/providers/Microsoft.Web/serverfarms/${PLAN}" \
  --subscription "${SUB}" \
  --metric "CpuPercentage" "MemoryPercentage" \
  --aggregation Average Minimum Maximum \
  --interval PT5M --start-time "${START}" --end-time "${END}" \
  --output json \
  > "${OUT_DIR}/cpu_ram_full_window.json"

wc -c "${OUT_DIR}/cpu_ram_full_window.json"
```

### Step 3 — Incidents file + build the report

Incident markers (vertical dashed lines on every panel) come from a plain-text file, not hardcoded — one per investigation, living in that event's own folder:

```
# <event folder>/incidents.txt
# ISO_TIMESTAMP|Label — one per line, blank lines and # comments ignored.
2026-09-13T23:55:00Z|Crash 23:55Z
```

**Optional third field for color-coding by type:** `ISO_TIMESTAMP|Label|Type`. Type is optional — an omitted or unrecognized value falls back to `"default"` (orange), so every pre-existing `incidents.txt` renders unchanged. Known types and their colors live in `INCIDENT_TYPE_COLORS` in the script (currently `default`/`escalation` = orange, `oom` = magenta) — add a new entry there for a new type rather than reusing an existing one for a visually-distinct category. The legend ("Incident type") only appears once, on the top response-time panel, even though every panel shows the markers — it's suppressed on the other panels to avoid repeating it six times. Example mixing types:

```
2026-09-20T21:15:00Z|OOM exception|oom
2026-09-20T21:40:00Z|Escalated to on-call|escalation
```

Then build the report. `--stream-id`/`--from`/`--to` are always required by the script, even when `--raw-csv` is also given — `--raw-csv` only skips the Graylog fetch step, it doesn't make those three optional (confirmed 2026-09-17: omitting them produces a hard usage error). Omit `--raw-csv` instead if you want the script to fetch step 1 itself rather than reusing an existing export:

```bash
python3 cloud/scripts/appservice_perf_report.py \
  --raw-csv "<event folder>/data/httplogs_raw_export.csv" \
  --stream-id "<stream id from the table above>" --from x --to x \
  --app-label "<App>-PRO" \
  --metrics-json "<event folder>/data/cpu_ram_full_window.json" \
  --incidents-file "<event folder>/incidents.txt" \
  --data-dir "<event folder>/data" \
  --report-dir "<event folder>/data"
```

**Defaults, if you omit `--data-dir`/`--report-dir`:** data → `cloud/scripts/data/`, HTML report → `cloud/report/` (both resolved relative to the script's own location, not your cwd). Both use fixed filenames, so repeated runs against the *default* dirs overwrite each other regardless of `--app-label` — always pass explicit `--data-dir`/`--report-dir` (pointing into the event's own folder, as above) when working a specific investigation. The bare defaults are for one-off/ad-hoc looks, not saved investigations.

Outputs:
- `<data-dir>/timetaken_by_path_aggregated.csv` — `bucket,path,referer,status,avg_ms,max_ms,count`, one row per (10-min bucket × endpoint × referer × status code).
- `<report-dir>/timetaken_report.html` — the linked report. 7 panels when `--metrics-json` is given (max response time, CPU %, RAM %, request volume, response codes, zoom bar, avg-response-time heatmap); 5 without it. Endpoint legend on the top panel filters every other panel; the zoom bar (drag to select, click to reset) re-scales every panel's time axis together. Embeds the full dataset inline (Altair/Vega-Lite default) — same "raw paste-back data, local-only" handling as the CSV/JSON inputs, never committed (see `.gitignore`).

`--all-paths` includes Kudu/Azure infra + SignalR negotiate + static files (excluded by default — only versioned `/api/v<N>/...` routes count as app traffic otherwise).

## Known Vega-Lite gotcha if you ever edit the script

A per-child `scale(domain=<selection>)` binding is **silently discarded** when that chart is layered (`+`) with siblings that reference the same x field — Vega-Lite recomputes a plain union-of-data-extents domain instead, with no error. The brush's own selection signal still fires correctly (visible via a live signal listener), but nothing listens to it, so dragging the zoom bar appears to do nothing. Confirmed via the compiled low-level Vega spec (`vegaLite.compile(spec).spec`, checking for a `domainRaw: {signal: ...}` on each scale, not just `domain`). Fix: apply the domain-bound X encoding **once**, at the combined-layer level (`(a + b + c).encode(x=zoomed_x)`), never on an individual child before layering — except any sibling that must span the full plot width with no x of its own (e.g. the Azure-timeout threshold rule), which has to stay **outside** that group as a separate layer added afterward.

## Constraints

- Output every command as a copy-paste block — never run `az`/`curl` directly, never tell the user to run something without showing the block.
- `GRAYLOG_API_KEY` as the env var in `curl -u`, never a literal token.
- All three CPU/RAM aggregations (`Average Minimum Maximum`) — a plain average-only pull silently drops real burst data.
- Read-only throughout (`az monitor metrics list`, Graylog export) — no `⚠️ ACTION` banner needed for this workflow.
