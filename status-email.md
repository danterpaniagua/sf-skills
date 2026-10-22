# Skill: Status Email

Cross-project skill: writes the email that shows **what Operations did** on a ticket and its subtasks (or, if the user asks, over a period), so the work is visible to the team, the PMs or whoever the user names. Works the same across `loyalty/`, `smartpedidos/`, `operations/` and `cloud/` — not tied to any single project's `*-sre-output` skill. Like `status-report`, it reads the event folders of the project it is pointed at; the project only changes the events it reads and the product names it uses.

## Invocation

`/status-email <project> [parent ticket | event folders | window]`

- `<project>`: `loyalty`, `smartpedidos`, `operations` or `cloud`.
- **Default scope: one parent ticket and its subtasks** (for example GITIN-2222 with GITIN-2229 and GITIN-2240). A period or an explicit list of events is allowed only when the user asks for it. If the scope is ambiguous, ask instead of guessing.
- Ask once for the **audience** (Operations team, PMs, directors) if the user has not said: it sets the granularity (see Rules).

## How it differs from the sibling outputs

| Output | Use it for |
|---|---|
| `status-report` | Short Detectado / Corregido / Pendiente update for one finding or a few tickets |
| Each project's `*-sre-output` emails | One incident or one proposal |
| `status-email` (this skill) | An **account of Operations' work on a ticket and its subtasks**, with attribution: what Operations did, what it reported to Development, what Development did |

## Template

The structure follows the plain-text email of `smartpedidos/events/20261016_gitin2240-concentrador-alerta-fatal/email.txt` (bold section titles written as `*Title:*`, hyphen bullets, a closing `Saludos.`), extended with a "Realizado" section. Write the output as `status-report.md` **in the parent ticket's own event folder** (this exact file name; the content stays a plain-text email, ready to paste) (the email belongs to that ticket; create a separate folder only for a period report the user asked for, with an `investigation.md` stub), **UTF-8 plain text** inside the `.md` file — not quoted-printable like the example, which is an export artifact.

Sections, in order:

1. `Asunto:` — one line with the project and the period.
2. Greeting and one neutral sentence saying what the email covers (the ticket and its subtasks, or the period). **Never state the real purpose in the text** — do not write that the email is meant to show, make visible or highlight Operations' work; the content speaks for itself.
3. `*Realizado:*` — one bullet per ticket (the parent first, then each subtask), grouped by theme only in a period report with more than about six tickets: `GITIN-XXXX (URL) — title: what Operations did`, with the state taken from the ticket's own "Resuelto" row or `Status:` line.
4. `*Reportado a Desarrollo:*` — only when Development owns a code change that Operations found or reported. Credit each side: Operations reported/analyzed, Development developed/is developing the fix. Omit the section if nothing applies.
5. `*Inconvenientes detectados:*` — what went wrong or surprised during the work, only if a source records it.
6. `*Pendientes:*` — open items with their owner.
7. `*Tareas post-despliegue:*` — only if a deployment happened or is planned and a source says so; proposals are written as proposals, but Operations' own ongoing tasks (monitoring after the fix, comparing before/after) are not proposals: write them as work Operations continues ("continuaremos con el monitoreo ..."), without claiming results the sources do not record.
8. `Saludos.` and the signature.

## Sources

For each event folder in scope read, before writing: `investigation.md` (the `Status:` line and the open items), `ops.md` (title, Resumen, "Resuelto" row, "Acciones Tomadas", "Acciones propuestas"), `ops-events.md` and any `email_*.md` when present. Never write a line from the memory of the conversation alone when the files exist and may have moved on. Verify that every event folder and ticket you cite exists on disk and carries the ID you write.

## Rules

- **Spanish.** Peer-to-peer tone for Operations/engineering readers; plain business language for PMs.
- **Voice: Operations is the sender.** Write in the first person plural ("reportamos", "creamos", "comparamos", "les compartimos"), never "Operaciones reportó" or "Operaciones compara" in the third person. Development is referred to as "Desarrollo" in the third person. The name "Operaciones" stays only where it is the name of a place, such as the Chat space.
- **Never estimate timelines**; no code, queries or config; no internal infrastructure or data-layer names (databases, collections, queues, resource IDs, file paths) for PMs and above; no names of people; never name internal sibling repositories.
- **Jira:** the real ticket URL as a Markdown link `[GITIN-1234](https://...)` on every mention of a ticket (the `Asunto:` line may keep the bare ID), never a bare ID alone and never a local repo path. A piece of work without a ticket is described without a link; do not invent an ID.
- **Never flatten a hedge.** "Parcial", "pendiente de validación", "propuesta" and "no resuelto" stay as written in the ticket. Causal links, planned actions and outcomes appear only as far as the source supports them.
- **Attribution.** Do not take credit for Development's work and do not hide Operations' part: state who reported, who analyzed and who developed.
- **User-stated facts** (for example "the fix will be deployed today", "the deployment was done", "no new container deaths so far") may appear, worded as planned or reported ("según nos informaron", "hasta ahora"), never as verified or resolved, and are listed back to the user in the reply so they can confirm them.
- **Granularity by audience:** Operations/engineering readers get per-server and per-component detail; PMs and above get service-level aggregates only.
- **PM / CTO audience: say what was done, not how it works.** One short line per ticket (what was done and its state), at most one key number per ticket, no mechanisms, thresholds, intervals, queries or rule names, and no explanation of how a control or alert operates. If a sentence explains how something works, delete it.
- No unsolicited impact statements, and no `(H#)`/`(C#)` markers.
- The objective of the skill (showing Operations' work) stays implicit: no phrases such as "para que quede a la vista", "para mostrar lo que hicimos" or "para dar visibilidad" in the email.

## Before finishing

Re-read the draft against the sources and list to the user: (1) which lines come from the user's own statements, (2) anything marked partial or unverified in the sources, (3) events in the period that were left out and why. This skill never sends the email.
