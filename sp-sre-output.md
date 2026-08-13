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
| Investigation notes (`investigation.md`) | English |
| Jira tickets (`events/YYYYMMDD_description/`) | Spanish |
| Tech debt files (`docs/tech-debt/**`) | English |
| Emails to Operations | Spanish |
| Emails to PMs | Spanish or English — match what the user requests |
| Findings files (`docs/*-findings.md`) | English |

## Event Folder Layout

Each ticket lives in `smartpedidos/events/YYYYMMDD_description/`, matching the layout used across `loyalty/`, `operations/`, and `cloud/` (see root `CLAUDE.md` → "Investigation Files (cross-project)"). Files inside are named by suffix only — no `YYYYMMDD_description_` prefix, the folder already disambiguates. Every incident ticket produces at minimum three files:

| File | Purpose |
|---|---|
| `investigation.md` | Working notes — English, freely editable, created first (see below) |
| `ops.md` | Main ticket — written in Spanish, only once findings have converged |
| `ops-events.md` | Running activity log — append-only work journal |

Additional files as needed: `scripts.sh`, `scripts.js`, `email_ops.md`, `email_pm.md`.

## Investigation File (`investigation.md`)

Create this **first**, before the main ticket exists. Its job is to absorb the messy part of an investigation — working theories about a root cause, reversed conclusions, dead ends — so the ticket never has to. Write in English. Unlike `ops-events.md`, this file is **not append-only** — rewrite sections in place as understanding changes; it should always reflect current best understanding, not a historical trail of every theory that got discarded.

Also serves as a resumption point: if investigation spans multiple sessions, read this file first before re-deriving anything from raw command/query output.

```markdown
# Investigation — YYYYMMDD_description

**Status:** in progress | converged — ready for ticket

## Confirmed facts
- Fact, with evidence reference (Cx from the scripts file, or file:line for source code)

## Current working theory
One paragraph — the best current explanation, stated as current belief, not final fact until Status is "converged".

## Ruled out
- Theory — why it's ruled out, with evidence reference. Keep brief; this exists so a theory doesn't get re-investigated, not to preserve narrative.

## Open questions / next steps
- ...
```

Once `Status: converged`, write the main ticket from this file's final state only — the ticket presents the converged understanding directly, with no "we previously thought X" language.

## Ops Events File (`ops-events.md`)

Append-only work journal. One entry per meaningful action: investigation step, finding, remediation applied, status update, or follow-up. Never edit past entries.

```markdown
# Eventos — YYYYMMDD_description

## YYYY-MM-DD HH:MM — <título corto>

Descripción del trabajo realizado, hallazgo o estado.
```

## Jira Ticket Structure

Main ticket goes in `smartpedidos/events/YYYYMMDD_description/ops.md`. Written in Spanish. Include the actual Jira ticket ID (see root `CLAUDE.md` → "External References") in the summary table once known — ask the user for it if not already provided.

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
| Ticket Jira | (ask the user if not already known — see root `CLAUDE.md` → "External References") |
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

- Written in **Spanish**.
- **Never estimate timelines.**
- **No code of any kind** — no JS snippets, no MongoDB queries, no config fragments. Technical detail belongs in Jira tickets.
- Reference the Jira ticket ID and service name — never a local repo path. Ask the user for the Jira ID first if not already known (see root `CLAUDE.md` → "External References").
- Describe the problem and proposed actions in plain technical language.

## Emails to PMs

- **Never estimate timelines.**
- **No code of any kind.**
- Describe impact in business terms: affected platforms, order flow disruption, volume at risk.
- Never mention internal SRE work: findings files, log antipatterns, refactor plans, access changes.
- "Próximos pasos" includes only actions visible to PMs (platform changes, service recovery steps). If only internal actions remain, omit the section entirely.
- **Never flatten hedged findings into certainty.** If the source ticket/investigation calls something circumstantial, proposed-but-undecided, or unconfirmed, the email must keep an equivalent hedge or omit the claim — never state it as settled fact just because the summary is shorter. Applies to causal links ("como consecuencia" implies confirmed causation — don't write it if the source only says "consistent with"), future actions ("vamos a hacer X" means it was actually decided, not merely proposed in an action-item list), and outcomes ("resuelto", "sin impacto", "volvió a la normalidad") — only state these if something was actually checked at that level, not inferred from a lower-level signal like CPU or task-restart counts.
- Do not reference MongoDB collection names, SQS queue names, or internal service names in PM emails.
- May mention the real Jira ticket ID for "more information" once it's confirmed to exist (see root `CLAUDE.md` → "External References") — that's a valid, clickable reference. Never use internal findings/script markers like `(H1)`/`(C3)` in a PM email, and never a local repo path — neither resolves to anything a PM could open.
