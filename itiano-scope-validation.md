---
name: scope-validation
description: Validate implementation against approved scope and detect scope creep.
---

# Mission

Verify that implementation matches approved scope.

Detect:

* Missing scope items
* Out-of-scope changes
* Repository rule violations
* Risks

---

# Inputs

Review:

* Scope document
* Modified files
* Acceptance criteria
* Repository rules
* CLAUDE.md requirements

---

# Validation Process

## Scope Coverage

For each scope item:

* Implemented
* Partially implemented
* Not implemented

## Scope Creep Detection

Identify:

* Unexpected file changes
* Additional features
* Refactors not included in scope

## Repository Compliance

Verify:

* CLAUDE.md compliance
* Architectural consistency
* UI standards compliance
* Repository conventions
* All changes are on the correct feature branch, not on `main`

---

# Output

# Scope Validation Report

## Completed Scope Items

## Partial Scope Items

## Missing Scope Items

## Out-of-Scope Changes

## Repository Compliance

## Risks

## Recommended Actions

---

# Reconciliation

If out-of-scope changes are found, stop and present them to the user. Ask explicitly:

For each out-of-scope change:

> "{change}" was not in scope. What should we do?
> 1. Fold into scope — document it in the scope file under `## Delivered`
> 2. Revert — list the exact files and lines to restore
> 3. Defer — create a new scope for it

Do not proceed until the user chooses one option per out-of-scope change.

## Folding into scope

When user selects option 1:

* Add a `## Delivered` section to the scope file if not present.
* Under `## Delivered`, add a subsection describing the change: what was done, which files, why it was added.
* Do not alter the original `## In Scope` or `## Technical Changes` sections — those reflect the approved plan.

## Reverting

When user selects option 2:

* List the specific files and the changes to undo.
* Do not revert automatically — wait for user confirmation before touching files.

## Deferring

When user selects option 3:

* Note the change as pending scope.
* Recommend a version number for a follow-up scope.
* Do not create the scope file until the user explicitly approves.

---

# Completion Rule

A task is complete only when:

* Scope items are implemented.
* Acceptance criteria are satisfied.
* All out-of-scope changes have been reconciled (folded, reverted, or deferred).
* Repository standards remain compliant.
