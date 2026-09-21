# Skill: Voice & Tense Check (ops-events.md)

Audits `ops-events.md` files (and legacy long-form `*_events.md` equivalents) for violations of the first-person voice/tense rule that every `*-sre-output` skill (`loyalty-sre-output`, `sp-sre-output`, `ope-sre-output`) requires for that file type. This command exists because recalling the rule from memory/skill text has repeatedly failed to prevent violations — only an actual grep pass over the written text has ever caught them. Run it as a mechanical check, not a read-through.

## The Rule

**Tense:** pretérito perfecto compuesto — "he verificado", "he confirmado", "he identificado". Not simple past ("verifiqué", "confirmé") and not simple present ("verifico", "confirmo").

**Exception — `loyalty` uses simple past instead, deliberately.** Confirmed 2026-09-21 (previously an open question, now settled): `loyalty-sre-output.md` requires pretérito perfecto **simple** ("confirmé", "detecté", "extraje"), not compound. `sp-sre-output` and `ope-sre-output` keep the compound tense above. When checking a loyalty `ops-events.md`, invert Stage 2: simple-past first-person verbs are **correct** there, not violations — instead check that compound-tense forms ("he confirmado") haven't crept in by mistake, since that would now be the actual inconsistency for that project.

**Person:** true first person. I am the one who investigates and acts. Never third person ("el usuario", "el operador") and never the impersonal passive ("se ha...", "se han...", "hemos...").

**No hierarchy/attribution framing:** never present an action or finding as triggered by someone else's order or question — "a pedido de...", "a pregunta de...", "ante el pedido de...", "he recibido la indicación/instrucción de...", "junto con el usuario/operador...". These still imply a commander/executor hierarchy even when "el usuario" isn't named. Rewrite as a direct statement of the action/decision/finding, with no external source attributed — e.g. "No toco el stream por ahora" (not "he recibido la indicación de no tocar..."), "He verificado si X existe, chequeo fuera de alcance de este ticket" (not "a pedido, he verificado si X existe").

**Legitimate exception:** stay passive/factual for an event I did not personally perform — e.g. "se reinició la VM" when the user ran the restart, not "he reiniciado la VM". This is accurate authorship, not a hierarchy claim, and is not a violation. Explaining *why* I didn't run something myself (e.g. a project rule against executing commands directly) is also fine: "con el usuario ejecutando el comando, no yo directamente" is a factual note, not third-person narration of my own work.

## Invocation

`/voice-check <path>` — `path` may be:
- a specific `ops-events.md` / `*_events.md` file
- a directory (checks every matching file under it, recursively)
- omitted — checks every `ops-events.md` / `*_events.md` file that was written or edited earlier in the current conversation

## Detection

Run both stages as actual tool calls — do not substitute a visual read-through, per the documented failure mode (recall alone has never caught this).

**Stage 1 — third-person / impersonal / attribution:**
```bash
grep -n -i "el usuario\|el operador\|se ha \|se han \|hemos \|a pedido\|a pregunta\|ante el pedido\|junto con\|indicación\|instrucción" <file>
```

**Stage 2 — for `sp`/`ope` (and cloud) files: simple-past or simple-present first-person verbs** (only meaningful for new/recently-touched entries, not full historical re-audits unless asked — see Scope below). Run all three of these, not just the first — each catches a different miss:
```bash
# capitalized/sentence-initial simple-past (é/í endings)
grep -noE '\b[A-ZÁÉÍÓÚ][a-záéíóú]*(é|í)\b' <file>
# lowercase mid-sentence simple-past (same endings, missed by the capitalized-only pattern above)
grep -noE '\b[a-záéíóú]*(é|í)\b' <file>
# simple-present first-person verbs — no accent, invisible to both patterns above
grep -noE '\b(Confirmo|Corrijo|Concluyo|Registro|Dejo|Verifico|Encuentro|Reviso|Analizo|Cierro|Reconfirmo)\b' <file>
```
All three over-match (proper nouns, non-verbs, "sí"/"ahí"/"esté" and similar non-verb é/í words, present-tense words used as nouns) — read each hit in context, don't trust the count. Look for section headers too, not just body prose; past occurrences of this bug hid in headers that a body-only pass missed. The present-tense word list is illustrative, not exhaustive — if a new first-person present-tense verb shows up in a hit from careful reading (not caught by the list), treat it as a real finding and add it to the list for next time.

**Stage 2 — for `loyalty` files: compound-tense creep, and simple-present, instead.** Simple past is correct here (see exception above) — don't flag it. Check for the opposite mistake instead:
```bash
# compound-tense forms that shouldn't be there in a loyalty file
grep -noE '\bhe [a-záéíóú]+ado\b|\bhe [a-záéíóú]+ido\b' <file>
# simple-present first-person verbs — still wrong in loyalty too (needs simple past, not present)
grep -noE '\b(Confirmo|Corrijo|Concluyo|Registro|Dejo|Verifico|Encuentro|Reviso|Analizo|Cierro|Reconfirmo)\b' <file>
```

## Fix Procedure

For each Stage 1 hit:
1. Check whether the legitimate exception applies (event genuinely not performed by me, and the sentence states that accurately). If so, leave it.
2. Otherwise rewrite as a direct first-person, hierarchy-free statement, preserving the factual content — don't just delete the flagged phrase if that would drop information (e.g. "fuera de alcance de este ticket" scope notes must survive the rewrite).

For each genuine Stage 2 hit in `sp`/`ope`/cloud files (first-person action verb narrating something I did): rewrite to "he/ha + participio". For a loyalty file, a genuine Stage 2 hit (compound-tense creep, or simple-present) rewrites the other direction: to simple past ("confirmé", not "he confirmado" or "confirmo").

After fixing, re-run both greps on the touched lines and confirm clean before reporting done.

## Draft correctly, don't fix after

The point of this check is a safety net, not a license to draft loosely and clean up later. Before writing any sentence into a `*ops-events.md`/`*_events.md` file, draft it mentally in "he + participio" (or simple past specifically for `loyalty` — see exception above), true first person, zero external attribution — the tense/person/attribution rules above, applied *before* the Edit/Write call, not after. Present-tense narration feels natural when describing something just done, and for `sp`/`ope`/cloud files simple past does too — that instinct is exactly what produces violations there, so don't trust the first phrasing that comes to mind for this file type specifically. For loyalty specifically, the equivalent trap is drifting into compound tense out of habit from the other projects.

Still run the grep passes after every write, every time, with no exception for "I drafted carefully this time" — recalling the rule has never been sufficient on its own (documented recurrence in `feedback_events_voice.md`, memory), but a session that only runs this check once at the end, after many writes, has failed the actual ask: getting it right the first time, not catching it after the fact.

## Scope

- Default target is `ops-events.md` / `*_events.md` files only — this is where the rule applies; do not extend the check to `ops.md` (future/imperative tense, a different rule) or `investigation.md` (English, not subject to this rule at all).
- When no path is given and nothing matching was touched this session, ask which file/event to check rather than scanning the whole repo — `events/` trees exist for loyalty, operations, smartpedidos, and cloud, and a full-repo pass without a target is expensive and usually not what's wanted.
- **Settled 2026-09-21** (previously an open question, listed here for history): `loyalty-sre-output.md`'s documented examples ("confirmé", "detecté", "extraje") are simple past, intentionally different from the compound-tense rule used in `sp`/`ope`. This is a confirmed, deliberate per-project divergence, not a bug — see "The Rule" above for the inverted Stage 2 check this implies for loyalty files.

## Output

Table: `file` | `line` | `original` | `fixed` | `rule violated`

Then state whether the post-fix grep came back clean. If a Stage 1 hit was judged to be the legitimate exception and left unchanged, list it separately as "reviewed, no change" with the reason — don't silently omit it.
