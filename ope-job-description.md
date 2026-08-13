# Skill: Job Description Sync

Keep the three role job-description documents in `operations/docs/` accurate against real, demonstrated capability — not aspirational or copy-pasted generic requirements.

## Target files

| File | Scope | Contains comparison tables? |
|---|---|---|
| `operations/docs/job_description_completa.md` | Combined Operations + SRE + DevOps — the recommended profile to publish unless Dirección splits the team | Yes — the only file with the "por qué un solo documento" note and the "División de foco por rol" stack table. Both must stay three-column (Operations / SRE / DevOps) since all three sibling docs exist. |
| `operations/docs/job_description_sre.md` | SRE only, trimmed | No — plain JD, one-line pointer to siblings only |
| `operations/docs/job_description_devops.md` | DevOps only, trimmed | No — plain JD, one-line pointer to siblings only |

There is no standalone `job_description_operations.md` — Operations-specific responsibilities live only inside `job_description_completa.md`.

Each file's shape: Resumen del puesto → Responsabilidades principales → Stack tecnológico → Requisitos excluyentes → Requisitos deseables → Seniority sugerido. Match existing section order and heading style exactly — don't restructure.

## When to run this

- After a session that surfaced new evidence of real capability (an incident investigated, a skill file discovered, a repo explored) that isn't yet reflected in these docs.
- When asked to "update the job descriptions" or similar, with or without a specific source named.

## Evidence sources — check all, don't guess

1. **`.claude/commands/*.md`** (and subdirectory-scoped `.claude/commands/` where they exist) across every repo in scope: `bots/` (root + any per-subproject commands), `cloud-graylog/`, `smartfran/sp-logs/`, and any other sibling repo the user names. Each skill file's stated scope is a claim about real capability — read it, don't infer from the filename alone.
2. **Actual session evidence** — a real incident investigated, a real file read, a real root cause found. A capability described in a skill file but never actually exercised is weaker evidence than something demonstrably done (e.g. the UberEats scoping-bug investigation was concrete evidence for "root-cause across app code + logs + infra + git history," not just a skill description).
3. **Project `CLAUDE.md` files** for stack/architecture details a skill file might reference but not spell out (DB names, cloud providers, service names).

Cross-check every candidate addition against what's *already* in the target doc(s) — don't duplicate an existing stack-table entry, extend it in place instead (see how `PostgreSQL (replicación streaming primary/replica)` became `PostgreSQL (administración completa: tuning, capacity planning, replicación streaming primary/replica, troubleshooting operativo)` rather than a second bullet).

## Deciding which file(s) a capability belongs in

Always add genuinely new capability to `job_description_completa.md` — it's the superset. Then judge fit for the trimmed docs by what actually drives the work, not by keyword association:

- **Reactive, root-cause, investigation-driven** (an incident, a "why did X happen" question, cross-layer diagnosis) → also add to `job_description_sre.md`.
- **Build/deploy/provisioning-driven** (a pipeline, IaC, a repeatable environment, a release) → also add to `job_description_devops.md`.
- **Schedule-driven, routine maintenance** (cert renewal, credential rotation, routine config with no design element) → `job_description_completa.md` only; there's no trimmed Operations doc to add it to.
- **Proactive but design-heavy work** (architecting a new monitoring/telemetry integration from scratch for a previously-unobserved critical service, not just flipping on a template) sits closer to Operations in trigger but is still real observability-*design* work — add to `job_description_sre.md` too, since both docs already carry a general "diseñar el stack de observabilidad" responsibility. Don't let the trigger-type heuristic override that overlap when the SRE doc already claims the broader design responsibility this is a concrete instance of.
- If a capability doesn't obviously map to a trigger type, ask rather than guess — this determines which files get edited, not just wording.

Don't force-fit something into a trimmed doc just because it touched the same technology. DBA depth (SQL Server / MongoDB Atlas / CosmosDB — event investigation, query optimization, index maintenance) went into SRE because root-cause investigation is what drives it, explicitly *not* into DevOps even though DevOps also touches those same database platforms operationally (provisioning, not administering).

## What to add, concretely

For each qualifying capability, touch — in the target file(s) — all of:
1. A **responsibility bullet** under the relevant subsection (`Responsabilidades principales`, or `Operación y mantenimiento` / `Confiabilidad e incidentes (SRE)` in `completa.md`), written as what the person actually does, not a tech buzzword list.
2. The **Stack tecnológico** table row for the relevant category — extend an existing row if the technology is already listed, add a new row only if the category doesn't exist yet.
3. In `completa.md` only: the matching cell in **"División de foco por rol"**, phrased as what that specific role does with that category (not a repeat of the stack list).
4. A **Requisitos excluyentes** or **Requisitos deseables** line if the capability is significant enough to gate a candidate on — not everything needs one.
5. The **Seniority sugerido** paragraph, only when the addition changes the *core identity* of the role (e.g. realizing the profile is fundamentally "SRE with real DBA depth" rather than "SRE that also touches databases") — not for every incremental stack addition.

## Language and tone

Spanish, matching the existing docs — these are real hiring documents, not translated from English. No sycophantic phrasing, no inflated buzzwords ("world-class," "rockstar"). Describe what the role actually does, grounded in evidence, the same evidentiary standard as an incident ticket.

## Guardrails

- Never invent a capability that isn't backed by a skill file's stated scope or actual demonstrated work in this repo's history.
- Don't duplicate content across the three files beyond what's structurally necessary (each trimmed doc repeats only what applies to *that* role, not the full combined text).
- If removing/narrowing a prior addition (e.g. "DBA yes, fraud investigation no" — a real correction from an earlier session), remove it from every file it was added to, not just the one most recently discussed — grep for the term across all three files to confirm full cleanup before reporting done.
- After edits, do a final `grep` pass across all three files for the term(s) involved to confirm the intended file set (and only that set) contains the addition.
