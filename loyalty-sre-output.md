# Skill: SRE Output

Produce formatted outputs (emails, reports, Jira tickets) for different audiences.

## Authorship

**Dante Paniagua, SRE** — peer member of the Operations Team, not a team lead or manager.

- Emails to Operations: peer-to-peer tone. Propose and coordinate — never direct or instruct.
- Reports for PMs: clear, non-technical language.
- Never use language that implies authority over the recipient.

## Event Folder Layout

Each event lives in `loyalty/events/YYYYMMDD_description/`. Files inside are named by suffix only — no `YYYYMMDD_description_` prefix, the folder already disambiguates. Every event produces at minimum three files:

| File | Purpose |
|---|---|
| `investigation.md` | Working notes — English, freely editable, created first (see below) |
| `ops.md` | Main ticket / closure report — written in Spanish, only once findings have converged |
| `ops-events.md` | Running activity log — append-only work journal |

Additional files as needed: `scripts.sql`, `queryXXX.sql`, `email_ops.md`, `email_pm.md`, `transferencias_pm.csv`.

## Investigation File (`investigation.md`)

Create this **first**, before `ops.md` exists. Its job is to absorb the messy part of an investigation — working theories about a fraud pattern or root cause, reversed conclusions, dead ends — so `ops.md` never has to. Write in English — the one exception to the project's rule that content written to `events/` is Spanish (`loyalty/CLAUDE.md`). Unlike `ops-events.md`, this file is **not append-only** — rewrite sections in place as understanding changes; it should always reflect current best understanding, not a historical trail of every theory that got discarded.

Also serves as a resumption point: if investigation spans multiple sessions (common for multi-day fraud tracing), read this file first before re-deriving anything from raw query output.

```markdown
# Investigation — YYYYMMDD_description

**Status:** in progress | converged — ready for ticket

## Confirmed facts
- Fact, with evidence reference (Qx from the .sql file, or CustomerId/table reference)

## Current working theory
One paragraph — the best current explanation (fraud mechanism, entry point, actor role), stated as current belief, not final fact until Status is "converged".

## Ruled out
- Theory — why it's ruled out, with evidence reference. Keep brief; this exists so a theory doesn't get re-investigated, not to preserve narrative.

## Open questions / next steps
- ...
```

Once `Status: converged`, write `ops.md` from this file's final state only — the ticket presents the converged understanding directly, with no "we previously thought X" language.

## Ops Events File (`ops-events.md`)

Append-only work journal. One entry per meaningful action: investigation step, query result, finding, status update, or follow-up. Never edit past entries.

**Voice:** first person throughout, pretérito perfecto — Dante narrating his own work ("confirmé", "detecté", "extraje"). Never third person ("el usuario pidió", "el usuario confirmó") and never impersonal ("se ha confirmado"). This applies even when narrating a request or a fact Dante provided in conversation — write it as his own action or finding, not as something reported to/by an external "usuario". Facts about external events (e.g. a VM restart) stay factual/passive if Dante didn't perform them himself (`se reinició la VM`, not `reinicié la VM`) — only the investigation actions are first person. Also never frame a decision/pivot as receiving an order from an external party ("he recibido la indicación de...", "a pedido de...") — that still implies a commander/executor hierarchy even without naming "el usuario". State the decision as a direct fact/action instead.

**Mandatory check:** after every Edit/Write to `ops-events.md`, invoke `/voice-check` on that file before treating the entry as done. Recalling this rule from memory alone has repeatedly failed to catch violations — only the mechanized grep in `voice-check.md` has.

```markdown
# Eventos — YYYYMMDD_description

## YYYY-MM-DD HH:MM — <título corto>

Descripción del trabajo realizado, hallazgo o estado.
```

## SQL Queries in Tickets

All SQL queries run during an investigation or fix must be saved to a `.sql` file in the event subfolder (`scripts.sql`). The ticket body references the file with a brief description table — no inline SQL blocks:

| # | Query | Propósito |
|---|---|---|
| Q1 | Short name | One-line description of what the query does |

For DBA investigations, trace query text comes from `PNSSRL_AuditSysprocesses.comando_ejecutado` and `PNSSRL_TempdbProc.Query_Text` — save those to `queryXXX.sql` as named in `dba-investigation`.

## Emails to Operations

- **Never estimate timelines.** Do not suggest how long a fix will take or when it will be deployed.
- **No code of any kind.** No SQL snippets, no script fragments, no config examples. Describe problems and fixes in plain language only. Technical detail belongs in Jira tickets.
- Reference the Jira ticket ID — never a local repo path (e.g. `loyalty/events/...`). Ask the user for the Jira ID first if not already known (see root `CLAUDE.md` → "External References").

## Emails to PMs

- **Never estimate timelines.**
- **No code of any kind** — except for database issues (see below).
- Never mention internal Operations matters: permissions, access reviews, security findings, or internal remediation.
- "Próximos pasos" includes only actions visible or relevant to PMs (infrastructure changes, service impacts).
- If the only next steps are internal, omit the section entirely — no explanation needed.
- **Never flatten hedged findings into certainty.** If the source ticket/investigation calls something circumstantial, proposed-but-undecided, or unconfirmed, the email must keep an equivalent hedge or omit the claim — never state it as settled fact just because the summary is shorter. Applies to causal links, future actions ("vamos a hacer X" means it was actually decided, not merely proposed), and outcomes ("resuelto", "sin impacto") — only state these if something was actually checked at that level, not inferred.
- May mention the real Jira ticket ID for "more information" once it's confirmed to exist (see root `CLAUDE.md` → "External References") — that's a valid, clickable reference. Never use internal findings/query markers like `(H1)`/`(Q3)` in a PM email, and never a local repo path — neither resolves to anything a PM could open.

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
- May mention the real Jira ticket ID for "more information" once confirmed to exist — never internal case/query markers, never a local repo path.

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

- Reference the Jira ticket ID — never a local repo path. Ask the user for the Jira ID first if not already known (see root `CLAUDE.md` → "External References").
- Describe the fraud pattern technically (fan-in, automation cadence, limit breach, etc.).
- No timelines. No code in the email body — evidence files carry the detail.

### Jira Tickets (Fraud)

**Tags:** always apply the `SmartLoyalty` project tag from `docs/tags.md`, plus any project-specific/cross-cutting tags that apply (e.g. `Fraude`, `SML-Puntos`).

Use exactly these sections in this order:

**Resumen** — one paragraph: what happened, when, which database/platform.

**Tabla resumen** — key event metadata:

| Campo | Valor |
|---|---|
| Ticket Jira | (ask the user if not already known — see root `CLAUDE.md` → "External References") |
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
