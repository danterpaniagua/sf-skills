# Skill: App Service Performance Report

Builds one linked HTML report (Altair/Vega-Lite) that correlates per-endpoint HTTP response time, request volume, response codes and App Service Plan CPU/RAM on one shared, zoomable timeline with incident markers, for any of the 8 production SmartFran Cloud App Services. The script is `cloud/scripts/appservice_perf_report.py` (born in `cloud/events/20260913_pos-net-http-timeout/`); this skill documents the 3-step workflow and the per-app values it needs.

## Scope

- The 8 production App Services: Admin, Business, Catalog, Orders, Person, Platform, Pos, Sales.
- Correlating slow/failing endpoints with App Service Plan CPU/RAM pressure.
- Not for DEV: the per-app `PROD-{App}-AppServiceHTTPLogs` streams below are PROD-only (the DEV stream layout is not documented).

## Known infrastructure

Subscription SmartIT Cloud (`85c76dea-3304-4310-8656-bf21b28e4f4b`), resource group `SmartFran.Cloud.PRO` (all 8 apps and plans), Graylog base URL `https://sfcloud-monitoreo.smartfran.com/graylog/api`. App Service / Plan names follow `SmartFran-Cloud-{App}-PRO` / `SmartFran-Cloud-{App}-Plan-PRO`; stream ids confirmed live 2026-08-26 (`cloud/events/20260826_appservicehttplogs-stream-split/investigation.md`):

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

**Pos caveat:** whether Pos emits `AppServiceHTTPLogs` at all is unconfirmed (`cloud/docs/graylog-log-fields.md`); ~0 rows from step 1 for Pos is that open question, not a workflow bug.

**Function Apps are out of scope** (`SFC-CodeGenerator-PRO`, `SFC-ScheduledJobs-PRO`, `SFC-PosWasmRelay-PRO`, `SFC-TicketProcess-PRO` emit `FunctionAppLogs`, not `AppServiceHTTPLogs`; this script's parser does not handle it). Two confirmed ways to get their traffic (`cloud/events/20260915_gitin2042-waf-cost-estimate/`, `cloud/events/20260915_functionapp-onboarding/`):
- **Aggregate, any window, no Graylog:** `az monitor metrics list --resource ".../Microsoft.Web/sites/<Function App>" --metric FunctionExecutionCount --aggregation Total --interval PT1H --start-time <ISO> --end-time <ISO>`: one number per app per hour, no per-function split.
- **Per function, Graylog, from 2026-09-15 only** (promotion is not retroactive): `category:"FunctionAppLogs" AND name:"<APP-UPPERCASE>"` (`name` is indexed uppercase; mixed case returns 0 silently), filter on `FunctionName` and `FunctionEventName:"FunctionCompleted"` (one row per real invocation, not per log line). Fields: `cloud/docs/graylog-log-fields.md` → `FunctionAppLogs`.

## Workflow — 3 steps, one report

Copy-paste blocks; the user runs them and pastes results back. Requires `$GRAYLOG_API_KEY` (never a literal token) and an authenticated `az` session.

### Step 1 — Graylog raw export (`CsUriStem`/`TimeTaken`/`ScStatus`/`Referer`)

Pulls the raw, unparsed `properties` JSON, bypassing field promotion (which is unreliable for historical data on some branches, `cloud/docs/graylog-log-fields.md` → HTTP request fields), so it works regardless of promotion status.

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

Request all three aggregations: a plain pull returns only Average and hides bursts (CPU averaged 15 % but peaked at 53 % within one 5-minute window). `PT10M` is **not** a valid Azure Monitor granularity (`PT1M/PT5M/PT15M/PT30M/PT1H/PT6H/PT12H/P1D` only); use `PT5M`, which divides into the script's 10-minute buckets.

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

Incident markers (vertical dashed lines on every panel) come from a plain-text file in the event's own folder:

```
# <event folder>/incidents.txt
# ISO_TIMESTAMP|Label — one per line, blank lines and # comments ignored.
2026-09-13T23:55:00Z|Crash 23:55Z
```

**Optional third field, the type:** `ISO_TIMESTAMP|Label|Type`; missing or unknown falls back to `default` (orange). Types and colors are `INCIDENT_TYPE_COLORS` in the script (`default`/`escalation` orange, `oom` magenta): add an entry there for a visually distinct category. The legend ("Incident type") appears once, on the top panel. Example:

```
2026-09-20T21:15:00Z|OOM exception|oom
2026-09-20T21:40:00Z|Escalated to on-call|escalation
```

Build the report. `--stream-id`/`--from`/`--to` are **always required**, even with `--raw-csv` (which only skips the Graylog fetch; omitting them is a hard usage error). Omit `--raw-csv` to let the script fetch step 1 itself:

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

**Defaults** without `--data-dir`/`--report-dir`: data → `cloud/scripts/data/`, report → `cloud/report/` (relative to the script, not the cwd), with fixed filenames, so repeated runs overwrite each other regardless of `--app-label`. For an investigation always pass explicit dirs inside the event folder; the defaults are for ad-hoc looks.

Outputs:
- `<data-dir>/timetaken_by_path_aggregated.csv` — `bucket,path,referer,status,avg_ms,max_ms,count`, one row per (10-min bucket × endpoint × referer × status code).
- `<report-dir>/timetaken_report.html` — the linked report: 7 panels with `--metrics-json` (max response time, CPU %, RAM %, request volume, response codes, zoom bar, avg-response-time heatmap), 5 without. The endpoint legend on the top panel filters every panel; the zoom bar (drag to select, click to reset) rescales all time axes. It embeds the full dataset: treat it like the raw CSV/JSON inputs, local only, never committed (`.gitignore`).

`--all-paths` includes Kudu/Azure infra + SignalR negotiate + static files (excluded by default — only versioned `/api/v<N>/...` routes count as app traffic otherwise).

## Reading the report — traps confirmed in solved events

Sources: `cloud/events/20260913_pos-net-http-timeout/` (GITIN-2040), `20260915_sales-appservice-perf-report/`, `20260920_sales-oom-48h/` (GITIN-2075).

- **Right stream.** `TimeTaken`/`CsUriStem` live only in `PROD-{App}-AppServiceHTTPLogs`; the `PROD-{App}-AppServicePlan` stream (the one most Sales queries use) had 32 of 1.3M messages with `ElapsedMilliseconds`. Check that the export's last timestamp reaches `END`: a stale export once gave a false "no traffic" (coverage ended 00:58Z).
- **`max_ms` ≈ 230,000 is the platform, not the code.** The Azure load balancer cuts an idle connection at 230 s on Windows (240 s on Linux), fixed, not configurable. A row at that value means the worker was unresponsive (GC/thread-pool stall) and whatever was in flight got cut: the endpoint name is incidental, so many endpoints hitting 230 s in one bucket is one process stall, not one hot path. Such rows can show `500` or no 5xx at all (0 of 640,105 rows in the 48 h OOM window): the client's own timeout fires first.
- **Incident markers need an independent source.** An endpoint cluster at 230 s is only a candidate crash. Cross-check with the App Service crash detector (`GET .../detectors/appcrashes`, Proactive Crash Monitoring; time range capped at 72 h, so split longer windows; it lists `0xE0434352` crashes and only two are inspectable as `OutOfMemoryException`) and with the `HealthCheckStatus` metric (PT5M or PT1M). In GITIN-2040 only one of three clusters had a health-check dip: more endpoints in a cluster does not mean a real crash, and no dip does not disprove one.
- **Memory per instance.** Plan-level `MemoryPercentage` is a blended average and hides one hot instance (one instance 87 % while two others were at 37-50 %): split by the `Instance` dimension (Portal; the exact CLI flag was not recorded in the events). Use PT1M for a narrow window; PT15M smoothed a sharper final spike.
- **`ComputerName` is the VM, not the worker process**: an OOM restart under ANCM keeps the same VM, so instance turnover cannot confirm or rule out a crash.
- **Empty App Insights / `AppExceptions` for the Windows apps is not evidence against an OOM**: the process dies before the SDK flushes.

## Known Vega-Lite gotcha if you ever edit the script

Layering coloured incident markers with a panel's own colour channel (endpoint / status / heatmap) merges the colour scales and breaks that panel's legend: set `resolve_scale(color="independent")` on every affected panel.



A per-child `scale(domain=<selection>)` is **silently discarded** when the chart is layered (`+`) with siblings referencing the same x field: Vega-Lite recomputes a union-of-extents domain, with no error, and the zoom bar appears to do nothing although its selection signal still fires. Verify on the compiled spec (`vegaLite.compile(spec).spec`: a `domainRaw: {signal: ...}` on each scale, not just `domain`). Fix: apply the domain-bound X encoding **once** at the combined-layer level (`(a + b + c).encode(x=zoomed_x)`), never on a child before layering; a sibling that must span the full width with no x of its own (e.g. the Azure-timeout threshold rule) stays **outside** that group as a layer added afterward.

## Constraints

- Output every command as a copy-paste block — never run `az`/`curl` directly, never tell the user to run something without showing the block.
- `GRAYLOG_API_KEY` as the env var in `curl -u`, never a literal token.
- All three CPU/RAM aggregations (`Average Minimum Maximum`) — a plain average-only pull silently drops real burst data.
- Read-only throughout (`az monitor metrics list`, Graylog export) — no `⚠️ ACTION` banner needed for this workflow.
