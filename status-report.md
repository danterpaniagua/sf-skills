# Skill: Status Report

Cross-project skill: produces a short-form status update for stakeholders (PM, ops leadership) from one or more tickets. Works the same way across `loyalty/`, `smartpedidos/`, `operations/`, and `cloud/` — not tied to any single project's own `*-sre-output` conventions, though it follows the same underlying discipline those skills already apply to PM-facing output.

## Invocation

`/status-report <path(s)>` — one or more event folders (`<project>/events/YYYYMMDD_description/`). If omitted, use whichever ticket(s) were the subject of the current conversation — ask which ticket(s) if that's ambiguous rather than guessing.

Read each ticket's `investigation.md` (current understanding) and `ops.md` (converged findings) before writing anything — never write a status line from memory of the conversation alone when the ticket files exist and may have moved on.

## When to use this vs. a full PM email

Use this when the user asks for a "status" update, says a detailed email is too long, or explicitly asks for this skill. It is a compressed sibling of each project's detailed PM/Operations email format, not a replacement — both can coexist for the same ticket.

## Output rules

Same discipline as any PM-facing output, restated here since this command is cross-project and can't assume every project's own skill file is loaded:

- **Spanish**, business language — no jargon a PM/ops leader wouldn't already use.
- **Never estimate timelines.**
- **No code, config, or query snippets of any kind.**
- Product/service names are fine to use directly (the reader manages that product and knows it by name) — but never internal infrastructure or data-layer names (database/collection names, queue names, resource IDs, internal file/service paths).
- **Never flatten a hedged finding into certainty.** If the source `investigation.md`/`ops.md` calls something proposed, pending, or unconfirmed, this report must keep an equivalent hedge — "en evaluación," "pendiente de decisión" — never state it as resolved just because the format is short. A short report is exactly where this mistake is easiest to make; being terse is not license to also be more certain than the source material.
- Only the real Jira ticket ID(s) as references — never a local repo path, never internal markers like `(H1)`/`(C3)`.

## Output sections, in order

1. **Asunto** — one line, ticket ID(s) included.
2. **Detectado** — one sentence: what was found.
3. **Corregido** — one sentence: what's actually fixed, not what's in progress.
4. **Pendiente** — one sentence: what's left, hedged per the rule above.
5. **Tabla de cambios** — only components that actually changed, nothing pending: `Proyecto | Componente | Antes | Después`. If nothing has actually changed yet, omit this section entirely rather than leaving it empty or filling it with pending items.
6. **Línea de impacto** — one line on the effect on the relevant operation (order flow, service availability, campaign delivery, etc. — whatever applies to the project), only if something was actually checked at that level, never inferred from a lower-level signal.
7. **Más información** — the Jira ticket URL(s), not just the bare ID — a PM should be able to click straight through. Nothing else in this line.
8. **Saludo** — plain sign-off, matching the sending project's usual tone (peer-to-peer for Operations, plain for PMs — see each project's own `*-sre-output` skill if one exists).

## Multiple tickets, one finding

When two or more tickets share the same underlying finding (sibling subtasks under the same parent epic, or the same incident affecting multiple components), produce **one consolidated report** referencing all the ticket IDs — not one near-duplicate per ticket. If a per-ticket file would otherwise exist, put the consolidated report in the most-advanced/active ticket's folder and leave a one-line pointer in the sibling ticket(s)' `investigation.md` so a future session doesn't look for a copy that isn't there.
