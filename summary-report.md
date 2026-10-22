# Skill: Summary Report

Cross-project skill: produces a multi-day work summary written by the user (an on-call shift, a weekend, a sprint week). Modeled on `operations/events/20261018_gitin2252-guardia-dia-de-la-madre/resumen.md`.

## Difference with `status-report`

| | `status-report` | `summary-report` |
|---|---|---|
| Scope | One particular problem (one ticket or sibling tickets) | The work of several days, across many items and tickets |
| Shape | Fixed short form: Asunto / Detectado / Corregido / Pendiente / table | Sections per project, one bold item per topic |
| Source | Ticket files (`investigation.md`, `ops.md`) | What the user tells during the session, plus the event folder |
| Output file | In the ticket's folder, per its own convention | `resumen.md` in the period's event folder |

## Invocation

`/summary-report <event folder>` — the folder of the period (e.g. a shift ticket). The period (first and last day) comes from the user; ask in plain text if it is missing.

The file is built incrementally: the user dictates events during the session and each one is added to `resumen.md` at once, in the right place. If the file exists, read it first and edit in place (extend an existing item rather than duplicating it).

## Voice

- Always written by the user, in the **first person** and in the **past tense**: what was done, found or measured is narrated as already happened (`he configurado`, `reinicié la cola a las 20:13`). Both pretérito perfecto and indefinido are valid, as in the model. Present tense only for a state that still holds now.
- Never attribute the user's actions to "SRE", "el equipo" or "lo informado", and never mention Claude.
- What was measured and what was not verified is stated plainly (`no he verificado`, `no sé por qué`).

## Events the user reports

- When the user says something happened, add it **as fact**, without "según", "reportado" or a verification label. This overrides the generic rule to validate user claims independently, for this skill only.
- First look for evidence in the tickets (the event folders' `investigation.md`, `ops.md`, `ops-events.md` and `data/`) and use it to place and complete the item (project, time, ticket, numbers). Evidence found there takes precedence over assumptions.
- When there is no evidence in the tickets, asking the user is allowed (project, time, ticket, ambiguity); if there is still no answer or evidence, treat the event as true and write it as such.
- Keep every hedge the user states themselves (`causa sin identificar`, `no sé si…`); never add certainty they did not give, and never invent a cause, a time or a number.

## Format

```
# Resumen <name> ([GITIN-NNNN](https://smartit-ar.atlassian.net/browse/GITIN-NNNN)) — del <weekday> dd-mm al <weekday> dd-mm

## <Project>
- **<Item> ([GITIN-NNNN](...)):** text.

## Cambios aplicados
- text, with ticket links.

## Pendiente
- text.

## Tickets
- <what it covers>: [GITIN-NNNN](...)
- Sin ticket todavía: <items without one>.
```

- One `## <Project>` per product that has items (SmartPedidos, SmartLoyalty, SmartFran Cloud, Operaciones…); omit empty ones.
- Each item is a bullet that starts with a bold label ending in a colon inside the bold, `**Label:**`. Put the ticket link in the label when the item has one.
- `## Cambios aplicados` holds what was changed on a system (alerts, thresholds, sizing), with the date; observations stay in the project sections.
- `## Pendiente` holds open causes, measurements still to take and checks still to do, hedged as the user stated.
- `## Tickets` lists every ticket cited, plus a `Sin ticket todavía` line for items that have none.
- Dates are `dd-mm` with the weekday only in the title range; times are 24h.

## Output rules

- **Spanish**, no code, config or query snippets.
- Ticket references are always markdown links, never a bare ID, never a local path, never `(H1)`/`(C3)` markers.
- Names of machines and services are allowed (Ops audience); for a PM audience the user will say so, and then it is service-level only.
- No impact statements or timelines unless the user gives them.
- Raw secrets, PII and endpoints are never written (see root `CLAUDE.md`).
