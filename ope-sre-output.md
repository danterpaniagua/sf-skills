# Skill: Operations SRE Output

Produce formatted event artifacts (Jira tickets, closure reports, emails) for infrastructure and cloud operations incidents.

## Authorship

**Dante Paniagua, SRE** — peer member of the Operations Team, not a team lead or manager.

- Emails to Operations: peer-to-peer tone. Propose and coordinate — never direct or instruct.
- Reports for PMs: clear, non-technical language.
- Never use language that implies authority over the recipient.

## Event Folder Layout

Each event lives in `operations/events/YYYYMMDD_description/`. Use the date the alert or incident was detected, not the date of investigation.

Every event produces at minimum three files:

| File | Purpose |
|---|---|
| `YYYYMMDD_description_ops.md` | Main ticket / closure report — written in Spanish |
| `YYYYMMDD_description_ops-events.md` | Running activity log — append-only work journal |
| `YYYYMMDD_description_scripts.sh` | All commands and scripts run during investigation and remediation |

Additional files as needed: `_scripts.py`, `_scripts.ps1`, `_email_ops.md`, `_email_pm.md`.

## Commands and Scripts in Tickets

All commands run during an investigation or remediation must be saved to the scripts file. The ticket body references the file with a brief description table — no inline command blocks in the ticket body:

| # | Comando / Script | Propósito |
|---|---|---|
| C1 | Short name | One-line description of what the command does |
| C2 | Short name | One-line description |

Mark any command that modifies state with `⚠️` in the Propósito column.

## Scripts File Format

```bash
#!/usr/bin/env bash
# Event: YYYYMMDD_description
# Commands are grouped by phase: Investigation / Audit / Remediation
# ⚠️ ACTION commands are clearly marked

# === INVESTIGATION ===
# C1 — <short name>
<command>

# === AUDIT ===
# C2 — <short name>
<command>

# === REMEDIATION ===
# ⚠️ C3 — <short name>
<command>
```

## Ops Events File (`_ops-events.md`)

Append-only work journal. One entry per meaningful action: investigation step, remediation applied, finding, status update, or follow-up. Never edit past entries.

**Tense:** pretérito perfecto impersonal, first person — "se ha verificado", "se ha identificado", "se ha confirmado". Yo soy quien ejecuta. Never refer to the author as "el usuario", "el operador", or any third-person subject.

```markdown
# Eventos — YYYYMMDD_description

## YYYY-MM-DD HH:MM — <título corto>

**Comando:** CX-N — <nombre corto>
**Resultado:** <output resumido>
**Observación:** Se ha <participio> — una línea de interpretación.
```

## Closure Report Structure (`_ops.md`)

Write in Spanish. This is a **Jira ticket describing work to be done** — use future or imperative tense throughout. Findings describe the current state; actions describe what must happen. Never write as if remediation is already complete.

Use exactly these sections in this order:

**Resumen** — one paragraph: what happened, when, which system/platform, severity.

**Tabla resumen** — key event metadata:

| Campo | Valor |
|---|---|
| ID alerta | |
| Sistema | |
| Severidad | |
| Detectado | |
| Resuelto | |
| Responsable | |

**Causa raíz** — one paragraph explaining the technical root cause.

**Hallazgos** — table of findings from the investigation:

| # | Hallazgo | Riesgo |
|---|---|---|
| H1 | Finding description | Alto / Medio / Bajo |

**Recursos afectados** — table of affected hosts, accounts, or services.

**Comandos ejecutados** — reference table pointing to the scripts file (no inline code):

| # | Comando / Script | Propósito |
|---|---|---|
| C1 | | |

**Acciones propuestas** — numbered list of actions taken or to be taken, each with outcome if completed. Section is always titled "Acciones propuestas" — never "Acciones requeridas".

**Hallazgos secundarios** — optional section for findings outside the primary scope that warrant follow-up.

## Emails to Operations

- **Never estimate timelines.**
- **No code of any kind** in the email body — scripts file carries the detail.
- Reference the Jira ticket and event subfolder.
- Describe the incident and proposed actions in plain technical language.

## Emails to PMs

- **Never estimate timelines.**
- **No code of any kind.**
- Describe impact in business terms: affected services, risk, proposed resolution.
- Never mention internal security findings, access reviews, or internal remediation details.
- "Próximos pasos" includes only actions visible or relevant to PMs. If only internal, omit the section.

## Language

All content written to `events/` must be in **Spanish**. All other conversational output in **English**.
