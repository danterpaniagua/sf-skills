# Skill: Context Sync

Post-session audit and update of all context files across the monorepo. Goal: keep guidance accurate, non-redundant, and token-efficient after work sessions introduce new facts, decisions, or structural changes.

---

## Files to Read

Read all of the following before starting:

**Root**
- `CLAUDE.md`
- `docs/*.md`
- `memory/MEMORY.md` and all `memory/*.md`
- `.claude/commands/*.md` — skill files (skim for cross-references and stale content only; do not rewrite skill logic)

**Sub-projects** (read each project's CLAUDE.md)
- `loyalty/CLAUDE.md` and `loyalty/docs/*.md`
- `operations/CLAUDE.md` and `operations/docs/*.md`
- `smartpedidos/CLAUDE.md` (if exists)
- `cloud/CLAUDE.md` (if exists)

Do not read from any `events/` directory.

---

## Checks and Actions

### A — Accuracy

**A1 — Stale facts**
Any file that states a version, IP, hostname, resource name, rule, or environment detail that was updated during the session must reflect the current value. Fix automatically.

**A2 — Cross-reference validity**
Any `../docs/`, `./docs/`, or skill file reference must point to a file that exists. Fix broken paths or remove the reference if the target was deleted.

**A3 — Memory vs. observed state**
Memory entries that name specific files, functions, flags, or infrastructure that have since changed must be updated. If a memory entry is confirmed stale, update or delete it.

---

### R — Redundancy

**R1 — Duplicated content across files**
If the same rule, fact, or table appears verbatim (or near-verbatim) in more than one file, keep it in the authoritative location only. Replace copies with a single-line reference: `See ../docs/<file>.md`.

**R2 — CLAUDE.md vs. skill overlap**
If a CLAUDE.md section re-states something already fully covered by a skill file, remove it from CLAUDE.md and add a pointer to the skill.

---

### T — Token Efficiency

**T1 — Verbose prose**
Any multi-sentence paragraph that restates what a table or list already conveys can be collapsed. Prefer tables and bullet lists over prose.

**T2 — Dead guidance**
Rules or notes that reference removed features, deprecated workflows, or past one-off decisions with no future relevance should be deleted.

**T3 — Example bloat**
Inline examples that are longer than necessary to illustrate the rule should be trimmed or removed if the rule is self-evident.

---

### C — Completeness

**C1 — New facts not yet documented**
If the session produced a new infrastructure fact (IP, NSG name, resource group, service name, environment detail), verify it appears in the appropriate reference doc under `docs/`. Add it if missing.

**C2 — New feedback not in memory**
If the session produced a user correction or confirmed preference that is not yet in `memory/`, write a memory entry following the existing format in `memory/MEMORY.md`.

**C3 — CLAUDE.md reflects current sub-project layout**
If directories, skills, or tools were added or removed during the session, update the relevant CLAUDE.md Directory Layout and Skills tables.

---

## Severity and Action

| Class | Action |
|---|---|
| **A** (Accuracy) | Fix automatically — wrong facts produce wrong behavior |
| **R** (Redundancy) | Fix automatically when the authoritative source is unambiguous |
| **T** (Token efficiency) | Fix automatically when meaning is fully preserved |
| **C** (Completeness) | Fix automatically for facts; flag for human review if context is ambiguous |

If a fix requires a judgment call about content (not just form), report it with a one-line proposal instead of applying it.

---

## Output Format

List each change applied:

```
[CLASS] <file> — <one-line description of what changed>
```

Then list items requiring human review:

```
[REVIEW] <file> — <one-line description of the question>
```

End with:
- Total files modified
- Total changes applied
- Total items flagged for review

If nothing required changes: output `Context files are current. No changes needed.`
