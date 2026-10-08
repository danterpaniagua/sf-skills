# Skill: DBA Investigation

Identify the query responsible for a database event given a time window.

## Investigation Workflow

1. User provides the event time window. Convert to GMT (+3h from UTC-3).
2. Output queries as copy-paste blocks — never execute them directly. User runs them on the server and pastes results back. **Every standalone query block needs its own `USE <database>;`** — don't rely on session/database context carrying over from a previous block (confirmed gap, GITIN-2036: two queries in the same `scripts.sql` were missing this and would have silently run against the wrong database or failed).
3. Run a data availability check on `PNSSRL_AuditSysprocesses` for the window.
4. Query capture tables using CPU delta per SPID between consecutive snapshots.
5. Cross-reference results to identify the responsible query/session.
6. Capture full query text from `PNSSRL_TempdbProc.Query_Text` for the responsible SPID. Provide it for the user to save as `queryXXX.sql` in the event subfolder before generating any output ticket.
7. **Always include `SET STATISTICS IO ON; SET STATISTICS TIME ON;`** (established 2026-09-25) in any query block whose cost/performance matters to the investigation — not just the fix-validation case below, any diagnostic query being run to understand why something is slow. Session-level, read-only, no elevated grant needed. Gives exact logical/physical reads per table and a clean CPU-time/elapsed-time split straight from SQL Server, instead of inferring cost from wall-clock timing or a live production capture. Turn back `OFF` at the end of the block, and note in the block's own comment that the output lands in the **Messages** tab (SSMS) or separately from the result set (`sqlcmd`) — not the results grid, so the user needs to paste both. Skip this only for a trivial single-row lookup where performance isn't in question.

## Validating a Proposed Query/Index Fix (established 2026-09-21)

Before proposing an index or query rewrite as a fix, capture a **before** baseline from the plan cache — not just CPU, but elapsed time and logical reads too (CPU time ≠ wall-clock elapsed time; a query can be I/O-bound with low CPU but high elapsed time):

```sql
SELECT
    qs.execution_count,
    qs.total_elapsed_time / 1000.0 AS total_elapsed_ms,
    qs.total_elapsed_time / qs.execution_count / 1000.0 AS avg_elapsed_ms_per_exec,
    qs.total_worker_time / qs.execution_count / 1000.0 AS avg_cpu_ms_per_exec,
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads_per_exec,
    qs.last_execution_time,
    qs.creation_time
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE st.text LIKE '%<table/column signature>%'
ORDER BY qs.total_worker_time DESC;
```

After applying an index (or before handing a rewritten query to Dev), **test the actual query with `SET STATISTICS IO ON; SET STATISTICS TIME ON;`** against real data — not just theory. Grab a real value for each search parameter first (e.g. `SELECT TOP 1 ... WHERE <col> IS NOT NULL`), then run the query being validated, comparing logical reads per table to the baseline. For a query with multiple branches (e.g. an `OR`, or a proposed `UNION` rewrite), test each branch in isolation with a real value — don't assume a shared index helps every branch equally. **Confirmed 2026-09-21 (GITIN-2074): an index alone showed zero improvement while the query's `OR` shape remained (still ~104,970 logical reads, matching the pre-index baseline, even on plans freshly compiled after the index existed) — only removing the `OR` in an isolated test proved the index worked fine on its own (104,970 → 4 logical reads for the same `CardNumber` value).** Re-running freshly-compiled plan-cache stats after the change (not just the manual test) also helps rule out "stale cached plan" as an alternative explanation for no improvement.

**Add the before/after comparison to `ops.md`**, not just `investigation.md` — as a `Hallazgo` (if it changes the diagnosis) or in `Acciones propuestas` (if it validates a fix). A logical-reads comparison translates directly into a PM-facing before/after table too — describe it as "how much of the table the database has to check," not "logical reads" (keep the PM-email no-jargon/no-code conventions from `loyalty-sre-output`).

## Capture Tables

| Table | Captured by | Useful for |
|---|---|---|
| `PNSSRL_AuditSysprocesses` | `PNSSRL_AuditoriaSysProcesses` | Full sysprocesses snapshot — primary source for CPU investigation |
| `PNSSRL_TempdbProc` | `PNSSRL - Captura_TEMPDB` | Sessions with tempdb usage + query text |
| `PNSSRL_CapturaQuery` | `PNSSRL_CapturaQuery` | Execution count + last execution time for Query078 |
| `PNSSRL_Usage_CpuMemory` | *(feeder disabled)* | CPU/memory — confirmed 2026-09-19: table is completely empty (0 rows, `MAX(fecha_hora_captura)` NULL), not just stale |
| `PNSSRL_DetallesEjecucionTablasModulos` | `PNSSRL_MantenimientoModulos` | Index maintenance execution detail — per-table/index `filas`, `fragmentacion`, `ejecucion`/`fin_reindexado`/`fin_indexdefrag`/`fin_ue` timestamps. `fin_reindexado` set = REBUILD ran; `fin_indexdefrag` set = REORGANIZE ran; neither set = skipped (fragmentation below threshold) |
| `PNSSRL_TablasModulos` | *(config, not a capture table)* | Per-table control row for `PNSSRL_MantenimientoModulos` — `indexado`/`defragmentacion`/`habilitado` flags, one row per table (whole-table granularity, no per-index or per-partition scope). `PNSSRL_MantenimientoModulos` itself takes `@base, @modulo, @tabla, @fillfactor` params (readable via `sys.parameters` even though the procedure body is `WITH ENCRYPTION`) |

Retention: **90 days** (except `PNSSRL_DetallesEjecucionTablasModulos`: 20 days).

## Key Columns

**`PNSSRL_AuditSysprocesses`** — timestamp: `fecha_hora_captura`

| Column | Notes |
|---|---|
| `spid` | Session ID |
| `cpu` | Cumulative CPU ms — use delta between snapshots, not raw value |
| `blocked` | Blocking SPID (0 = not blocked) |
| `lastwaittype` / `waitresource` | Current wait |
| `status` | `running`, `runnable`, `sleeping`, `suspended` |
| `cmd` | Command type |
| `loginame` / `hostname` / `program_name` | Session identity |
| `comando_ejecutado` | Query text at capture time (varchar 8000) |
| `dbid` | Resolve with `DB_NAME(dbid)` |

**`PNSSRL_TempdbProc`** — timestamp: `hora_captura`

| Column | Notes |
|---|---|
| `session_id` / `request_id` | Session identity |
| `Query_Text` | Full query text (nvarchar MAX) |
| `cpu_time` / `logical_reads` / `reads` / `writes` | Resource usage |
| `OutStanding_user_objects_page_counts` | Tempdb user object pages |
| `OutStanding_internal_objects_page_counts` | Tempdb internal object pages (sorts, spills) |
| `granted_query_memory` | Memory grant pages |
| `HOST_NAME` / `login_name` / `program_name` | Session identity |

## OS-Level Diagnostics (beyond `PNSSRL` capture tables)

When a DB-host CPU/OS-level question (Windows Update, Windows Defender, Scheduled Tasks, `logman`/Performance Monitor, Resource Monitor) can't be answered from `PNSSRL` and needs a PowerShell command run directly on a server, **never assume which host the user's RDP session is actually on.** Multiple hosts are in play (`SFCG-DB01`, `SFCG-TO-01`, web/mobile app servers) and RDP access does not necessarily default to the one relevant to the investigation.

**Confirmed incident (2026-08-03, `GITIN-1749`):** an entire battery of OS-level checks (Windows Update history, Defender scan/detection, Scheduled Task inventory, `logman` Data Collector Sets) was run under the assumption of `SFCG-DB01`, but the user's RDP session was actually on `SFCG-TO-01` the whole time. Went undetected until a `logman`/`relog` CSV export exposed `\\SFCG-TO-01\...` counter prefixes — every prior "ruled out" conclusion had to be redone from scratch on the correct host.

**Rule:** every PowerShell command block for OS-level diagnostics must open with a hostname stamp, e.g.:
```powershell
"Running on: $env:COMPUTERNAME"
```
Treat the returned hostname as the ground truth for which host the rest of that command's output describes — don't rely on which host you *intended* to target. If the user hasn't confirmed the hostname recently in the conversation, ask before trusting new output, especially when switching between checking `SFCG-DB01` (the SQL Server engine) vs. `SFCG-TO-01` (TaskOperatorService, hosts its own `\SmartFran\` Scheduled Task folder — see `loyalty/docs/infrastructure.md`).

## Source Code Reference — check before concluding "no config exists"

When a question is about **scheduling/recurrence behavior or business logic** (not raw SQL Server internals), DB schema/data alone can lead to a confident-but-wrong "no lever exists" conclusion — check `loyalty/repo/dev-src-sol-smartloyalty/` (local read-only source clone, see `loyalty/docs/infrastructure.md` → "Source Code Reference") **early**, not as a last resort.

**Confirmed incident (GITIN-2036, 2026-09-14):** investigating why a `Sch.Job` campaign task ran 24/7 with no quiet hours, the claim "no way to reschedule/pause these jobs at the DB config level" was written into a doc based on `Sch.Job`'s own columns alone (no recurrence-interval column there) — and had to be corrected **twice** before finally being resolved: first when `Sch.Task.TimeTableId` was noticed pointing to an unexamined `Sch.TimeTable` config table (real `FrecuencyType`/`FrecuencyInterval` columns existed — DB permission denied on first read), then fully once the actual scheduling engine source (`Front/TaskOperatorService/TaskOperatorService.cs`) was read directly, which definitively answered both the interval question and a related "can two schedule rows coexist" question that the DB schema alone could not answer with any confidence. The same session also resolved a real vs. duplicate point-crediting concern by reading `Query084-sql.xml` directly rather than inferring behavior from watermark columns.

**Rule of thumb**: if a DB-only investigation is about to conclude "not possible" / "no config for that" regarding recurrence, business rules, or idempotency — grep the local repo clone for the relevant class/query name (often visible in `TaskEvent`/exception stack traces, e.g. `SmartFran.Solution.SmartLoyalty.Service.MktCampaignContext.MktAsignationPointActionService`) before writing that conclusion down. Note: some namespaces (e.g. `SmartFran.Business.Schedule.*`) live in a shared library **not present** in this clone — if a grep comes up empty, that's a real gap to flag honestly, not proof the behavior doesn't exist.

## Known Infrastructure

### Application servers

| Host | Role |
|---|---|
| `SFCG-WEBS-01/02/03` | WebService (`C:\SmartFran\SmartLoyalty.WebService\`) |
| `SFCG-WSV2-01` | WebServiceV2 (`D:\SmartLoyalty.WebServiceV2\`) |
| `SFCG-WSIT-01` | WebSite (`D:\SmartLoyalty.WebSite\`) |
| `SFCG-MOBI-01/02` | Mobile service |
| `SFCG-WSCG-01` | CG web service |
| `SFCG-CLUB-01/02` | Club Grido website |
| `SFCG-TO-01` | TaskOperatorService (`E:\SmartLoyalty.TaskOperatorService\`) |
| `SFCG-DB01` | SQL Server host — SQL Server 2022 Standard 64-bit, v16.0.4075.1, Windows Server 2022 Datacenter |
| `LUCAS-KIUVI` | QA workstation — must not connect to production |

### Database accounts

| Account | Type | Used by | Role on SmartLoyalty |
|---|---|---|---|
| `SMARTIT\itservices` | Windows domain | All app servers | cross-role (not audited) |
| `sfsqlusr` | SQL login | TaskOperatorService (SFCG-TO-01) | `db_owner`, `db_securityadmin` ⚠️ |
| `sfsqlusrit` | SQL login | QA / SSMS (LUCAS-KIUVI) | `db_datareader` — confirmed 2026-09-19, no longer over-privileged (previously `db_owner`/`db_securityadmin`) |
| `NT SERVICE\SQLSERVERAGENT` | Windows service | SQL Agent jobs | system |

⚠️ `sfsqlusr` still over-privileged (confirmed 2026-09-19).
