# Skill: Technical Debt

Record a technical debt item identified during analysis. Appends a row to the central log and creates an expanded explanation file.

**Language:** All output from this skill — log rows and expanded explanation files — must be written in **English**.

## Project labels

| Label | Service |
|---|---|
| `concentrador` | concentrador-service |
| `platform` | platforms-service |

## Log file

`docs/tech-debt.md` — central running table. Never overwrite; always append a new row.

## Expanded explanation file

`docs/tech-debt/<date>_<slug>.md` — one file per debt item.

- `<date>`: `dd-mm-yyyy`
- `<slug>`: 3–5 word kebab-case description (e.g. `branch-srp-violation`, `missing-ttl-index`)

## Workflow

### Step 1 — Collect

Gather from the user or from current analysis context:
- **Project**: `concentrador` or `platform`
- **File**: repo-relative path (e.g. `api/src/controllers/branch.js`)
- **Lines**: line range or total line count relevant to the debt
- **Summary**: one sentence, max 15 words
- **Root cause**: what created the debt and why it persists
- **Impact**: operational, observability, or maintenance impact if left unaddressed
- **Remediation**: recommended fix with effort estimate (Low / Medium / High)
- **References**: related findings, Jira tickets, commits, or SUB-XXX items

### Step 2 — Create expanded explanation file

Create `docs/tech-debt/<date>_<slug>.md` with this structure:

```markdown
# <summary>

**Project:** concentrador | platform
**File:** <path>
**Lines:** <range or count>
**Recorded:** <date>

## Root cause

## Impact

## Remediation

**Effort:** Low | Medium | High

## References
```

Ask before writing: "Do you want me to record this technical debt?"

### Step 3 — Append to log

Add one row to the table in `docs/tech-debt.md`:

```
| concentrador \| platform | `file` | lines | Summary sentence | [slug](tech-debt/date_slug.md) |
```

Never edit existing rows. Append only.
