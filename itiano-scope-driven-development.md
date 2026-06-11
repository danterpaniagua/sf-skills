---
name: scope-driven-development
description: Analyze requirements, define scope, manage versions, control implementation scope, and prepare delivery.
---

# Mission

Help the user stay focused on approved scope and prevent uncontrolled changes.

Never jump directly into implementation.

Always:

1. Analyze
2. Scope
3. Version
4. Implement
5. Validate
6. Deliver

---

# Core Principles

* Scope before code.
* Small scopes over large scopes.
* Explicit approval before implementation.
* No silent scope expansion.
* Repository rules always apply.
* CLAUDE.md rules always apply.

---

# Phase 1 - Requirement Analysis

Identify:

* Goal
* Business value
* Scope
* Out of scope
* Risks
* Dependencies
* Affected components

Ask questions if requirements are ambiguous.

Output:

## Requirement Analysis

### Goal

### In Scope

### Out of Scope

### Risks

### Dependencies

### Affected Components

### Open Questions

---

# Scope Splitting Rule

If the request contains multiple independent objectives:

* Do not create a single scope.
* Split the work into multiple scopes.
* Propose a version for each scope.
* Explain dependencies.
* Ask which scope should be executed first.

Prefer multiple small scopes over one large scope.

---

# Phase 2 - Version Definition

Propose a semantic version:

* Patch → vX.Y.Z
* Minor → vX.Y.0
* Major → vX.0.0

Explain why.

---

# Phase 3 - Scope Creation

Create:

.claude/vX.Y.Z.md

Include:

# Goal

# In Scope

# Out of Scope

# Technical Changes

# Acceptance Criteria

# Validation Plan

# Related Scope Dependencies

---

# Phase 4 - Branch Creation

After approval:

1. Run `git branch` to confirm current branch.
2. If not already on the correct branch: `git checkout -b vX.Y.Z`
3. Run `git branch` again to confirm the switch succeeded.

Never work directly on main. Never edit files before confirming the active branch.

---

# Phase 5 - Implementation

Before the first file edit, run `git branch` to confirm you are on the feature branch. If on `main`, stop and switch first.

Implement only approved scope items.

If new requirements appear:

1. Stop.
2. Update scope.
3. Request approval.
4. Continue only after approval.

---

# Phase 6 - Delivery

Before closing:

* Update scope document.
* Verify scope completion.
* Verify repository compliance.
* Run tests inside Docker: `docker compose exec app python manage.py test` — do not claim tests passed without actual output.
* Update `README.md` if new apps, env vars, or commands were added.
* Update `CLAUDE.md` if new commands or test instructions are needed.
* Move scope file from `.claude/vX.Y.Z.md` to `docs/vX/vX.Y.Z.md`.
* Do not output a commit message unless the user explicitly asks for one.

Never:

* Commit
* Push
* Merge

without explicit approval.

---

# Language

All responses, scope documents, and output must be in English.

# Suggested Commit Message

* Always write in Spanish.
* Never include `Co-Authored-By` trailers.
* Format: `tipo: descripción corta` followed by a blank line and body if needed.
