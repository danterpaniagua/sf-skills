# Skill: SRE Output

Produce formatted outputs (Jira tickets, emails, reports) for SmartPedidos incidents and engineering work.

## Authorship

**Dante Paniagua, SRE** — peer member of the Operations Team, not a team lead or manager.

- Emails to Operations: peer-to-peer tone. Propose and coordinate — never direct or instruct.
- Reports for PMs: clear, non-technical language.
- Never use language that implies authority over the recipient.

## Language policy

| Output | Language |
|---|---|
| Jira tickets (`docs/jira/**`) | Spanish |
| Tech debt files (`docs/tech-debt/**`) | English |
| Emails to Operations | English |
| Emails to PMs | Spanish or English — match what the user requests |
| Findings files (`docs/*-findings.md`) | English |

## Event Folder Layout

Each ticket lives in `docs/jira/dd-mm-yyyy_<small_title>/`. Every incident ticket produces at minimum two files:

| File | Purpose |
|---|---|
| `ticket.md` | Main ticket — written in Spanish |
| `ops-events.md` | Running activity log — append-only work journal |

Additional files as needed: `scripts.sh`, `scripts.js`, `email_ops.md`, `email_pm.md`.

## Ops Events File (`ops-events.md`)

Append-only work journal. One entry per meaningful action: investigation step, finding, remediation applied, status update, or follow-up. Never edit past entries.

```markdown
# Eventos — dd-mm-yyyy_<small_title>

## YYYY-MM-DD HH:MM — <título corto>

Descripción del trabajo realizado, hallazgo o estado.
```

## Jira Ticket Structure

All tickets go in `docs/jira/dd-mm-yyyy_<small_title>/ticket.md`. Written in Spanish.

**Tags:** always apply the `SmartPedidos` project tag from the root `docs/tags.md`, plus any project-specific/cross-cutting tags that apply (e.g. `Concentrador`, `Platform`, `MongoDB`).

### Log Improvements ticket (`dd-mm-yyyy_<service>-log-improvements`)

Use exactly these sections in this order:

**Descripción** — one paragraph: which service, what logging problems were found, scope of the cycle.

**Criterios de aceptación** — one bullet per SUB-XXX item addressed. Format:
- `[SUB-000]` Descripción del criterio cumplido.

**Sub-tareas** — table with columns `Sub-tarea`, `Archivo`, `Líneas`, `Descripción`. One row per fix.

**Estado final** — table with columns `Sub-tarea` and `Estado` (`✅ Resuelto` / `⚠️ Pendiente` / `❌ Descartado`).

**Log responsibility (follow-up)** — include only if SUB-010 findings exist. Describe the feasibility assessment and effort estimate. Mark as a separate follow-up story if effort is Medium or High.

---

### SRP Refactor ticket (`dd-mm-yyyy_<service>-srp-refactor`)

Use exactly these sections in this order:

**Descripción** — current state: file name, line count, cluster count, coupling issues.

**Sub-tareas** — one per cluster extraction, ordered by Value desc / Risk asc. Columns: `Cluster`, `Archivo destino`, `Líneas`, `Esfuerzo`, `Riesgo`.

**Criterios de aceptación**:
- Cada archivo extraído tiene una única responsabilidad.
- El archivo original reduce a menos de 500 líneas.
- Ningún export existente se rompe.

**Observabilidad (follow-up)** — include only if observability separation is deferred. Reference the event emitter pattern.

---

### Incident ticket (`dd-mm-yyyy_<service>-<incident>`)

Use exactly these sections in this order:

**Resumen** — one paragraph: what happened, when, which service and collection affected.

**Tabla resumen** — key event metadata:

| Campo | Valor |
|---|---|
| ID alerta | |
| Sistema | |
| Severidad | |
| Detectado | |
| Resuelto | |
| Responsable | |

**Causa raíz** — one paragraph.

**Hallazgos** — table of findings. Reference log entries, MongoDB collection names, SQS queue names, or error messages. No inline code blocks — reference findings files:

| # | Hallazgo | Riesgo |
|---|---|---|
| H1 | | Alto / Medio / Bajo |

**Recursos afectados** — table of affected services and collections:

| Componente | Impacto |
|---|---|
| platforms-service | |
| concentrador-service | |

**Comandos ejecutados** — reference table pointing to the scripts file (no inline code):

| # | Comando / Script | Propósito |
|---|---|---|
| C1 | | |

**Acciones propuestas** — numbered list. Each action references the responsible team (Dev / SRE / Infra).

**Hallazgos secundarios** — optional section for findings outside the primary incident scope.

---

## Emails to Operations

- **Never estimate timelines.**
- **No code of any kind** — no JS snippets, no MongoDB queries, no config fragments. Technical detail belongs in Jira tickets.
- Reference the Jira ticket folder and service name.
- Describe the problem and proposed actions in plain technical language.

## Emails to PMs

- **Never estimate timelines.**
- **No code of any kind.**
- Describe impact in business terms: affected platforms, order flow disruption, volume at risk.
- Never mention internal SRE work: findings files, log antipatterns, refactor plans, access changes.
- "Próximos pasos" includes only actions visible to PMs (platform changes, service recovery steps). If only internal actions remain, omit the section entirely.
- Do not reference MongoDB collection names, SQS queue names, or internal service names in PM emails.
