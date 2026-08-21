# Skill: Operations SRE Output

Produce formatted event artifacts (Jira tickets, closure reports, emails) for infrastructure and cloud operations incidents.

## Authorship

**Dante Paniagua, SRE** — peer member of the Operations Team, not a team lead or manager.

- Emails to Operations: peer-to-peer tone. Propose and coordinate — never direct or instruct.
- Reports for PMs: clear, non-technical language.
- Never use language that implies authority over the recipient.

## Event Folder Layout

Each event lives in `operations/events/YYYYMMDD_description/`. Use the date the alert or incident was detected, not the date of investigation.

Every event produces at minimum four files:

| File | Purpose |
|---|---|
| `investigation.md` | Working notes — English, freely editable, created first (see below) |
| `ops.md` | Main ticket / closure report — written in Spanish, only once findings have converged |
| `ops-events.md` | Running activity log — append-only work journal |
| `scripts.sh` | All commands and scripts run during investigation and remediation |

Additional files as needed: `scripts.py`, `scripts.ps1`, `email_ops.md`, `email_pm.md`.

Files are named by suffix only — no `YYYYMMDD_description_` prefix, the folder already disambiguates.

## Investigation File (`investigation.md`)

Create this **first**, before `ops.md` exists. Its job is to absorb the messy part of an investigation — working theories, reversed conclusions, dead ends — so `ops.md` never has to. Write in English (the one exception to the events/ Spanish rule below). Unlike `ops-events.md`, this file is **not append-only** — rewrite sections in place as understanding changes; it should always reflect current best understanding, not a historical trail of every theory that got discarded.

Also serves as a resumption point: if investigation spans multiple sessions, read this file first before re-deriving anything from raw command output.

```markdown
# Investigation — YYYYMMDD_description

**Status:** in progress | converged — ready for ticket

## Confirmed facts
- Fact, with evidence reference (Cx from scripts file, or file:line for source code)

## Current working theory
One paragraph — the best current explanation, stated as current belief, not final fact until Status is "converged".

## Ruled out
- Theory — why it's ruled out, with evidence reference. Keep brief; this exists so a theory doesn't get re-investigated, not to preserve narrative.

## Open questions / next steps
- ...
```

Once `Status: converged`, write `ops.md` from this file's final state only — the ticket presents the converged understanding directly, with no "we previously thought X" language. Findings that were ruled out along the way don't appear in the ticket at all unless they're independently useful context (e.g. ruling out the WAF as a cause is worth stating in the ticket; the fact that three different VM-reactivation theories were tried before the right one is not).

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

## Graylog Pipeline Mitigations — Validation Requirement

When a fix takes the form of a Graylog Pipeline Rule attached to a stream, a simulator pass alone is not sufficient confirmation — the simulator only proves the rule's logic is correct given a message already routed to that stream, it does not prove the stream actually receives the traffic the fix targets. Before writing "confirmed"/"mitigado" anywhere in `investigation.md` or `ops.md`, verify the stream's own routing rules (`GET /api/streams/{id}/rules`) actually match the traffic in question — do not infer stream scope from a source-code match (e.g. "this service's code matches, so its stream must be the one routing this index's traffic") without checking. Then confirm on real live traffic the same way any other fix in this project is confirmed (a doc-count delta, an `_exists_` search, or absence of the previously-seen indexer failure) before treating the mitigation as validated.

Confirmed gap (GITIN-1854, 2026-08-15): a `msg_rest_status` sanitization rule passed cleanly in the simulator, but the underlying stream attachment (`SP_Concentrador`) was never checked against its actual routing rules — inferred instead from a source-code match. The same failure class continued on live traffic within the hour, undetected until the next round of indexer-failure evidence was pasted back.

## Ops Events File (`ops-events.md`)

Append-only work journal. One entry per meaningful action: investigation step, remediation applied, finding, status update, or follow-up. Never edit past entries.

**Tense:** pretérito perfecto, true first person — "he verificado", "he identificado", "he confirmado". Yo soy quien ejecuta. Never refer to the author as "el usuario", "el operador", or any third-person subject, and never use the impersonal "se ha..." construction. Also never frame a decision/pivot as receiving an order from an external party ("he recibido la indicación de...", "a pedido de...") — that still implies a commander/executor hierarchy even without naming "el usuario". State the decision as a direct fact/action instead: "No toco el stream por ahora" / "He retomado X", not "He recibido la indicación de no tocar/retomar X".

**Mandatory check:** after every Edit/Write to `ops-events.md`, invoke `/voice-check` on that file before treating the entry as done. Recalling this rule from memory alone has repeatedly failed to catch violations — only the mechanized grep in `voice-check.md` has.

```markdown
# Eventos — YYYYMMDD_description

## YYYY-MM-DD HH:MM — <título corto>

**Comando:** CX-N — <nombre corto>
**Resultado:**
<output>

<párrafo en primera persona, pretérito perfecto, sin etiqueta en negrita — interpretación del resultado, puede ser multi-oración>
```

Omit `**Comando:**` entirely when the entry documents a manual step (no real command/script ran) — go straight from the heading to `**Resultado:**`.

## Closure Report Structure (`ops.md`)

Write in Spanish. This is a **Jira ticket describing work to be done** — use future or imperative tense throughout. Findings describe the current state; actions describe what must happen. Never write as if remediation is already complete.

**Tags:** always apply exactly one project tag from `docs/tags.md` (`Operaciones` or `SmartCloud`, depending on which project the ticket belongs to) plus any project-specific/cross-cutting tags that apply — see that file for the full list.

Use exactly these sections in this order:

**Resumen** — one paragraph. Open with the problem/gap that existed *before* any work started — not with what was done — then which system/platform, severity, when detected. This framing doesn't change based on whether the ticket is drafted before or after the work: even a ticket written entirely after the fact, for planned/proactive work with no incident trigger, must state the motivating problem or objective first (e.g. "N apps lacked X, leaving Y exposed") before describing what was implemented. The chronological account of what was actually done belongs in `ops-events.md` — don't let this paragraph collapse into a "here's what we did" report just because the work is already finished.

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
- Reference the Jira ticket ID — never a local repo path (e.g. `operations/events/...`, `cloud/events/...`). Ask the user for the Jira ID first if not already known (see root `CLAUDE.md` → "External References").
- Describe the incident and proposed actions in plain technical language.
- **Exception for architecture/design proposal emails** (not incident-closure emails): when the email's purpose is to get a peer to evaluate a design trade-off — not to report an incident's status — Mermaid diagrams comparing current vs. proposed flow, and brief `file:line`/controller-name references, are acceptable. The reader needs enough to actually evaluate the trade-off, which plain prose alone often can't carry for an architecture change. The "no code" rule still applies in full to incident-status/closure emails, where the ticket already carries that detail and the email should stay pure narrative.

## Emails to PMs

- **Never estimate timelines.**
- **No code of any kind.**
- Describe impact in business terms: affected services, risk, proposed resolution.
- Never mention internal security findings, access reviews, or internal remediation details.
- "Próximos pasos" includes only actions visible or relevant to PMs. If only internal, omit the section.
- **Never flatten hedged findings into certainty.** If the source ticket/investigation calls something circumstantial, proposed-but-undecided, or unconfirmed, the email must keep an equivalent hedge or omit the claim — never state it as settled fact just because the summary is shorter. Applies to causal links ("como consecuencia" implies confirmed causation — don't write it if the source only says "consistent with"), future actions ("vamos a hacer X" means it was actually decided, not merely proposed in an action-item list), and outcomes ("resuelto", "sin impacto", "volvió a la normalidad") — only state these if something was actually checked at that level, not inferred from a lower-level signal.
- May mention the real Jira ticket ID for "more information" once it's confirmed to exist (see root `CLAUDE.md` → "External References") — that's a valid, clickable reference. Never use internal findings/command markers like `(H1)`/`(C3)` in a PM email, and never a local repo path — neither resolves to anything a PM could open.

## Language

All content written to `events/` must be in **Spanish**, with one exception: `investigation.md` is written in **English** (see above). All other conversational output in **English**.
