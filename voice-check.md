# Skill: Voice & Tense Check (ops-events.md)

Audits `ops-events.md` files (and legacy long-form `*_events.md` equivalents) for violations of the first-person voice/tense rule that every `*-sre-output` skill (`loyalty-sre-output`, `sp-sre-output`, `ope-sre-output`) requires for that file type. This command exists because recalling the rule from memory/skill text has repeatedly failed to prevent violations — only an actual grep pass over the written text has ever caught them. Run it as a mechanical check, not a read-through.

## The Rule

**Tense:** pretérito perfecto compuesto — "he verificado", "he confirmado", "he identificado". Not simple past ("verifiqué", "confirmé") and not simple present ("verifico", "confirmo").

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

**Stage 2 — simple-past or simple-present first-person verbs** (only meaningful for new/recently-touched entries, not full historical re-audits unless asked — see Scope below):
```bash
grep -noE '\b[A-ZÁÉÍÓÚ][a-záéíóú]*(é|í)\b' <file>
```
This over-matches (proper nouns, non-verbs) — read each hit in context, don't trust the count. Look for section headers too, not just body prose; past occurrences of this bug hid in headers that a body-only pass missed.

## Fix Procedure

For each Stage 1 hit:
1. Check whether the legitimate exception applies (event genuinely not performed by me, and the sentence states that accurately). If so, leave it.
2. Otherwise rewrite as a direct first-person, hierarchy-free statement, preserving the factual content — don't just delete the flagged phrase if that would drop information (e.g. "fuera de alcance de este ticket" scope notes must survive the rewrite).

For each genuine Stage 2 hit (first-person action verb narrating something I did): rewrite to "he/ha + participio".

After fixing, re-run both greps on the touched lines and confirm clean before reporting done.

## Scope

- Default target is `ops-events.md` / `*_events.md` files only — this is where the rule applies; do not extend the check to `ops.md` (future/imperative tense, a different rule) or `investigation.md` (English, not subject to this rule at all).
- When no path is given and nothing matching was touched this session, ask which file/event to check rather than scanning the whole repo — `events/` trees exist for loyalty, operations, smartpedidos, and cloud, and a full-repo pass without a target is expensive and usually not what's wanted.
- Known open question (unresolved, do not silently decide): `loyalty-sre-output.md`'s own documented examples ("confirmé", "detecté", "extraje") are simple past, inconsistent with the compound-tense rule applied elsewhere. If a loyalty `ops-events.md` is being checked, flag simple-past hits but note this discrepancy explicitly in the report rather than unilaterally normalizing loyalty to the cloud/operations tense convention — that's a cross-skill decision for the user to make once, not something to resolve file-by-file.

## Output

Table: `file` | `line` | `original` | `fixed` | `rule violated`

Then state whether the post-fix grep came back clean. If a Stage 1 hit was judged to be the legitimate exception and left unchanged, list it separately as "reviewed, no change" with the reason — don't silently omit it.
