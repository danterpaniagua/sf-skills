# Skill: DBA Investigation

Identify the query responsible for a database event given a time window.

## Investigation Workflow

1. User provides the event time window. Convert to GMT (+3h from UTC-3).
2. Output queries as copy-paste blocks — never execute them directly. User runs them on the server and pastes results back.
3. Run a data availability check on `PNSSRL_AuditSysprocesses` for the window.
4. Query capture tables using CPU delta per SPID between consecutive snapshots.
5. Cross-reference results to identify the responsible query/session.
6. Capture full query text from `PNSSRL_TempdbProc.Query_Text` for the responsible SPID. Provide it for the user to save as `queryXXX.sql` in the event subfolder before generating any output ticket.

## Capture Tables

| Table | Captured by | Useful for |
|---|---|---|
| `PNSSRL_AuditSysprocesses` | `PNSSRL_AuditoriaSysProcesses` | Full sysprocesses snapshot — primary source for CPU investigation |
| `PNSSRL_TempdbProc` | `PNSSRL - Captura_TEMPDB` | Sessions with tempdb usage + query text |
| `PNSSRL_CapturaQuery` | `PNSSRL_CapturaQuery` | Execution count + last execution time for Query078 |
| `PNSSRL_Usage_CpuMemory` | *(feeder disabled)* | CPU/memory — not receiving new data |
| `PNSSRL_DetallesEjecucionTablasModulos` | `PNSSRL_MantenimientoModulos` | Index maintenance execution detail |

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
| `sfsqlusrit` | SQL login | QA / SSMS (LUCAS-KIUVI) | `db_owner`, `db_securityadmin` ⚠️ |
| `NT SERVICE\SQLSERVERAGENT` | Windows service | SQL Agent jobs | system |

⚠️ Both over-privileged.
