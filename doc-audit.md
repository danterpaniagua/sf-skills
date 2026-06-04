# Skill: Documentation & Context Integrity Audit

Act as a documentation and context integrity auditor for this project.

## Scope

Read every guidance and reference file listed below, then perform a cross-document consistency audit. Fix all Critical and High discrepancies automatically. For Medium and Low, fix them if the correction is unambiguous; otherwise report and explain.

---

## Files to Read

Read all of the following before starting any checks:

1. `CLAUDE.md`
2. `.claude/commands/fraud-points.md`
3. `.claude/commands/dba-investigation.md`
4. `.claude/commands/sre-output.md`
5. `.claude/commands/azure-nsg.md`
6. All files under `docs/`
7. All `.md` files in the memory directory (`~/.claude/projects/-home-dpaniagua-Documentos-git-smartfran-loyalty-bot/memory/`)

Do not read from `events/` unless a specific check requires it and it is explicitly noted below.

---

## Consistency Checks

### Critical — wrong behavior if inconsistent

**C1 — EventTypeCode catalog**
Every file that lists EventTypeCode values must contain the same set with identical descriptions. Reference: `fraud-points.md` is the authoritative source. Check: `docs/`, memory files.

**C2 — Business rule limits**
Accumulation limits (daily 3,000 / weekly 5,000 / monthly 10,000 pts) and transfer limits (daily 8,000 / weekly 10,000 / monthly 13,000 pts) must be identical wherever stated. Reference: `fraud-points.md`. Check: `docs/`.

**C3 — Database access scope per skill**
`CLAUDE.md` global restriction: `PNSSRL` is the default; `SmartFran.Solution.SmartLoyalty` requires explicit request or implicit skill grant. Each skill must correctly declare its database scope:
- `fraud-points.md` → `SmartFran.Solution.SmartLoyalty` (implicit within skill)
- `dba-investigation.md` → `PNSSRL` only
- `sre-output.md` → no direct DB access
- `azure-nsg.md` → no DB access

**C4 — Transfer pairing key**
All files that describe transfer pairing must reference `sml.PointsTransference` as the primary key — not `LogDate` matching. LogDate match and sequential Id are secondary confirmation only.

**C5 — Table column lists**
Column lists for shared tables (`sml.CustomerPointsLog`, `sml.Person`, `sml.Customer`, `sml.PointsTransference`, `smlst.CustomerPointsLog`) must be identical across every file that declares them.

---

### High — incorrect outputs or misleading guidance

**H1 — Closure report format**
`CLAUDE.md` Output section and `fraud-points.md` Evidence Package (Accumulation anomaly closure) must specify the same three required sections: (1) metrics table, (2) EventTypeCode breakdown, (3) customer detail table. Column names and descriptions must match.

**H2 — Timezone references**
SQL Server timezone is GMT (UTC+0). User local timezone is UTC-3. All skills must apply the same conversion direction consistently. Specifically: `dba-investigation.md` states "+3h from UTC-3" to get GMT — verify this is arithmetically consistent with the direction used in `fraud-points.md` queries.

**H3 — Fraud patterns catalog**
The fraud patterns table in `fraud-points.md` and `docs/` must be identical in content. Extra patterns in `docs/` not present in `fraud-points.md` are inconsistencies.

**H4 — Investigation workflow alignment**
Q-numbers referenced in `fraud-points.md` accumulation workflow (Q1–Q8) must be consistently numbered and described wherever referenced across all files.

**H5 — Output language rule**
All content written to `events/` must be in Spanish. All other content (analysis, queries, skill outputs) in English. Check that no skill produces Spanish output for non-`events/` content, and that no skill instructs English-only output for `events/` files.

---

### Medium — reduced usefulness or stale guidance

**M1 — Memory vs. current skill content**
Each memory entry must not contradict current skill content or `CLAUDE.md`. Check: does any memory rule reference a workflow step, column name, or constraint that has since changed?

**M2 — Docs status markers**
Any document with status "Proposal" or pending checklist items that have since been implemented must be updated to reflect the current active state.

**M3 — Known infrastructure currency**
`dba-investigation.md` lists application servers and database accounts. Flag any entry marked with a status note (e.g., "feeder disabled", "⚠️ over-privileged") to confirm it is still current and accurately described.

**M4 — Cross-file references**
Explicit cross-references between files (e.g., "see `sre-output` for column spec") must point to a section that exists and contains the referenced information.

---

### Low — naming and formatting

**L1 — Event naming convention**
Any file that specifies event subfolder or file naming must use the canonical form from `CLAUDE.md`: subfolder `events/YYYYMMDD_description/`, files `YYYYMMDD_description_audience.ext`. Flag deviations.

**L2 — Read-only constraint scope**
`CLAUDE.md` prohibits DML/DDL against any database. `azure-nsg.md` prohibits write Azure CLI commands unless explicitly requested. Verify these are not narrower or broader than intended.

---

## Severity Classification

| Severity | Criteria |
|---|---|
| **Critical** | Inconsistency that would produce wrong queries, miss fraud signals, or grant incorrect database access |
| **High** | Inconsistency that would produce incorrect outputs or actively misleading guidance |
| **Medium** | Stale or incomplete information that reduces document accuracy or usefulness |
| **Low** | Naming, formatting, or minor wording with no operational impact |

---

## Output Format

For each discrepancy:

```
[SEVERITY] — <file(s)> — <check ID>: <description of the inconsistency>
Action: Fixed | Cannot fix automatically — <reason>
```

After all discrepancies: print a summary table:

| Severity | Found | Fixed | Reported only |
|---|---|---|---|
| Critical | | | |
| High | | | |
| Medium | | | |
| Low | | | |

End with overall status: **CONSISTENT** or **INCONSISTENCIES FOUND**.

If no discrepancies are found for a check, do not mention it — only report findings.
