# Skill: Context Sync

Post-session audit and update of all context files across the monorepo. Goal: keep guidance accurate, non-redundant, token-efficient, and portable (works for anyone who clones this repo, on any machine, at any path — see A5) after work sessions introduce new facts, decisions, or structural changes.

---

## Files to Read

Read all of the following before starting:

**Root**
- `CLAUDE.md`
- `docs/*.md`
- Claude Code's built-in **auto-memory system** — a per-project directory under `~/.claude/projects/<this-project's-path-with-slashes-replaced-by-dashes>/memory/` (the exact path is machine- and clone-location-specific; Claude Code's own per-session instructions state the resolved path for the current checkout — use that, don't hardcode a literal path here). Read its `MEMORY.md` and every `*.md` file in that directory. This lives outside the repo, under the user's home directory, not in git — and is the **only** memory store. A separate repo-tracked `memory/` folder existed briefly (2026-07-10 through 2026-08-30) as an unmaintained legacy duplicate whose content had drifted stale/contradictory relative to this one; deleted 2026-08-30 (`/context-sync`) once that was confirmed. If a `memory/` folder is ever found back at repo root in a future audit, treat its reappearance as worth asking about, not silently trusting.
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
Memory entries (in the auto-memory store, primarily) that name specific files, functions, flags, or infrastructure that have since changed must be updated. If a memory entry is confirmed stale, update or delete it.

**A4 — A repo-tracked `memory/` folder reappearing**
The repo-tracked legacy `memory/` folder was deleted 2026-08-30 after being found to contain content contradicting the auto-memory store (see the note in "Files to Read" above). If one exists again at repo root on a future audit, don't assume it's a duplicate to silently merge or delete — flag it (`[REVIEW]`) and ask why it's back before doing anything to it.

**A5 — Portability: machine-specific absolute paths**
`CLAUDE.md`, `docs/*.md`, and `.claude/commands/*.md` must work for anyone who clones this repo to any path, on any machine — none of them should hardcode something true only for the current checkout. Run this before finishing any audit:

```bash
git grep -nE "/home/[a-z_][a-z0-9_-]*/|~/\.claude/projects/-[a-zA-Z0-9-]+/" -- 'CLAUDE.md' '*/CLAUDE.md' 'docs/*.md' '*/docs/*.md' '.claude/commands/*.md'
```

A hit here means a literal username, home directory, or resolved auto-memory project-hash path got written into a file that's supposed to be portable — found and fixed once already (2026-08-30, `context-sync.md` itself hardcoded `~/.claude/projects/-home-dpaniagua-Documentos-git-bots/memory/`, which only resolves correctly on that one clone location). Fix by describing the thing generically instead (e.g. "Claude Code's per-project auto-memory directory — see the harness's own per-session instructions for the resolved path" rather than the literal path). This check does **not** apply to `events/` — those are point-in-time artifacts of what actually happened on a real machine, not portable instructions, and are excluded from context-sync reads anyway. Also verify any cross-reference this pass touches (e.g. "see root `CLAUDE.md` → X") actually resolves to a real heading in that file — a harness-injected system-prompt section (like the built-in auto-memory feature explanation) is not the same as a heading inside the actual `CLAUDE.md` file on disk, and citing it as if it were is itself an A2 violation.

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
If the session produced a user correction or confirmed preference that is not yet captured, write a memory entry in the **auto-memory store** (see "Files to Read" above for how to find its path), following the existing format in its `MEMORY.md`. There is no longer a competing repo-tracked `memory/` folder to avoid (see A4).

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
