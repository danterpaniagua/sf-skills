# Skill: SRE Output

Produce formatted outputs (emails, reports, Jira tickets) for different audiences.

## Authorship

**Dante Paniagua, SRE** — peer member of the Operations Team, not a team lead or manager.

- Emails to Operations: peer-to-peer tone. Propose and coordinate — never direct or instruct.
- Reports for PMs: clear, non-technical language.
- Never use language that implies authority over the recipient.

## Event Folder Layout

Each event lives in `loyalty/events/YYYYMMDD_description/`. Every event produces at minimum two files:

| File | Purpose |
|---|---|
| `YYYYMMDD_description_ops.md` | Main ticket / closure report — written in Spanish |
| `YYYYMMDD_description_ops-events.md` | Running activity log — append-only work journal |

Additional files as needed: `_scripts.sql`, `_queryXXX.sql`, `_email_ops.md`, `_email_pm.md`, `_transferencias_pm.csv`.

## Ops Events File (`_ops-events.md`)

Append-only work journal. One entry per meaningful action: investigation step, query result, finding, status update, or follow-up. Never edit past entries.

```markdown
# Eventos — YYYYMMDD_description

## YYYY-MM-DD HH:MM — <título corto>

Descripción del trabajo realizado, hallazgo o estado.
```

## SQL Queries in Tickets

All SQL queries run during an investigation or fix must be saved to a `.sql` file in the event subfolder (`YYYYMMDD_description_scripts.sql`). The ticket body references the file with a brief description table — no inline SQL blocks:

| # | Query | Propósito |
|---|---|---|
| Q1 | Short name | One-line description of what the query does |

For DBA investigations, trace query text comes from `PNSSRL_AuditSysprocesses.comando_ejecutado` and `PNSSRL_TempdbProc.Query_Text` — save those to `YYYYMMDD_description_queryXXX.sql` as named in `dba-investigation`.

## Emails to Operations

- **Never estimate timelines.** Do not suggest how long a fix will take or when it will be deployed.
- **No code of any kind.** No SQL snippets, no script fragments, no config examples. Describe problems and fixes in plain language only. Technical detail belongs in Jira tickets.

## Emails to PMs

- **Never estimate timelines.**
- **No code of any kind** — except for database issues (see below).
- Never mention internal Operations matters: permissions, access reviews, security findings, or internal remediation.
- "Próximos pasos" includes only actions visible or relevant to PMs (infrastructure changes, service impacts).
- If the only next steps are internal, omit the section entirely — no explanation needed.

### Database issues — exception

When the email concerns a database performance or availability issue, include:
- The responsible query (full text).
- Where it came from: host, application name, and login (`hostname` / `program_name` / `loginame` from `PNSSRL_AuditSysprocesses`, or `HOST_NAME` / `program_name` / `login_name` from `PNSSRL_TempdbProc`).

## Fraud Investigation Outputs

### Emails to PMs

PMs are the primary contact with Grido (the client) and need full incident detail to act on and forward directly.

- Describe impact in business terms: affected volume, timeline, fraud patterns in plain language.
- **No code of any kind.** Do not reference internal table names or query results.
- Never mention internal remediation, access changes, or security controls.
- Always attach `05_YYYYMMDD_transferencias_pm.csv` — include it as a reference in the email body.
- "Próximos pasos" includes only actions visible to PMs (e.g. platform controls, service changes). If the only next steps are internal, omit the section.

Include two summary tables in the email body — **anomalous accounts only**:

**Tabla 1 — Emisores con actividad anómala**: senders who exceeded the daily transfer limit (8,000 pts). Columns: `Nombre`, `DNI`, `Puntos enviados`, `Canal`.
**Tabla 2 — Receptores con actividad anómala**: receivers who received more than the daily transfer limit (8,000 pts) in a single day. Columns: `Nombre`, `DNI`, `Puntos recibidos`, `Canal`.

- Only include transactions that fall within the reported incident interval. Transactions detected outside that window (via the ±2h investigation padding) do not appear in the tables.
- Aggregate totals per customer across all their transfers within the incident interval.
- Accounts with no valid identity: use `Sin registro de identidad` in name and DNI.
- If a customer appears in both tables, include them in both.
- Channel: list all channels used by that customer in the event (e.g. `APP / WEB`).
- Other irregular patterns (fan-in, circular transfers, rapid reforward) are described in the intro paragraph — they do not generate additional table rows.
- Full participant list is in `05_YYYYMMDD_transferencias_pm.csv`.

### PM Evidence CSV (`05_YYYYMMDD_transferencias_pm.csv`)

One row per transfer. Columns:

`transfer_id, fecha_hora_local, emisor_nombre, emisor_dni, receptor_nombre, receptor_dni, puntos, canal, observacion`

- `fecha_hora_local`: UTC-3, formatted as `YYYY-MM-DD HH:MM`.
- `observacion`: plain-language fraud flag for suspicious rows; blank for clean transfers. Use labels like `Límite diario superado`, `Transferencia duplicada`, `Transferencia circular`, `Acumulación y reenvío inmediato`, `Fan-in`, `Identidad no resuelta`.
- Accounts with no valid Person record: use `[Sin registro de identidad]` in name and DNI fields.

### Emails to Operations

- Reference the Jira ticket and evidence subfolder (`events/YYYYMMDD_fraude_evidencia/`).
- Describe the fraud pattern technically (fan-in, automation cadence, limit breach, etc.).
- No timelines. No code in the email body — evidence files carry the detail.

### Jira Tickets (Fraud)

Use exactly these sections in this order:

**Resumen** — one paragraph: what happened, when, which database/platform.

**Tabla resumen** — key event metadata:

| Campo | Valor |
|---|---|
| Caso | |
| Base de datos | |
| Severidad | |
| Detectado | |
| Resuelto | |
| Responsable | |

**Causa raíz** — one paragraph explaining the fraud mechanism and entry point.

**Hallazgos** — table of detected fraud patterns:

| # | Hallazgo | Riesgo |
|---|---|---|
| H1 | | Alto / Medio / Bajo |

**Recursos afectados** — table of affected accounts, hubs, or relays.

**Métricas del evento** — table with columns `Métrica` and `Valor`. Include: investigation window (GMT), transfer count, total points, dominant channel, senders over daily limit, points involved in breaches, receivers with post-event activity, accounts registered by systemic registrar (if applicable).

**Consultas ejecutadas** — reference table pointing to the .sql file (no inline SQL):

| # | Query | Propósito |
|---|---|---|
| Q1 | | |

**Acciones propuestas** — numbered list of actions taken or to be taken.

**Archivos de evidencia** — table with columns `Archivo` and `Contenido`. List all files under `events/YYYYMMDD_fraude_evidencia/`.

**Hallazgos secundarios** — optional section for findings outside the primary fraud scope.

- Do not include personal data (names, DNIs) in the ticket body — reference the CSV files instead.
