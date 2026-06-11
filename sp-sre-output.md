# Skill: SRE Output

Produce formatted outputs (emails, reports, Jira tickets) for different audiences.

## Authorship

**Dante Paniagua, SRE** — peer member of the Operations Team, not a team lead or manager.

- Emails to Operations: peer-to-peer tone. Propose and coordinate — never direct or instruct.
- Reports for PMs: clear, non-technical language.
- Never use language that implies authority over the recipient.

## Trace Data

Always include trace data in Operations Jira tickets:
- Store the full captured query text as a `.sql` file inside the issue subfolder (`events/YYYYMMDD_description/`).
- Reference it by filename in the Jira ticket.
- Source columns: `PNSSRL_AuditSysprocesses.comando_ejecutado`, `PNSSRL_TempdbProc.Query_Text`.

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
**Patrones identificados** — table with columns `Patrón` and `Detalle`. One row per pattern detected.
**Métricas del evento** — table with columns `Métrica` and `Valor`. Include: investigation window (GMT), transfer count, total points, dominant channel, senders over daily limit, points involved in breaches, receivers with post-event activity, accounts registered by systemic registrar (if applicable).
**Archivos de evidencia** — table with columns `Archivo` and `Contenido`. List all files under `events/YYYYMMDD_fraude_evidencia/`.

- Do not include personal data (names, DNIs) in the ticket body — reference the CSV files instead.
