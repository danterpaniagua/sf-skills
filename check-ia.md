# Skill: Check IA (signs of AI-generated writing)

Audits a text file (an email, a ticket body, a report) for the signs of AI-generated writing catalogued in Wikipedia's [Signs of AI writing](https://en.wikipedia.org/wiki/Wikipedia:Signs_of_AI_writing) (WP:AISIGNS). It is built to be **token-efficient**: the first stages are plain `grep` counts that cost almost no tokens, and the model reads only the lines they flag. Run the stages as real tool calls, not as a read-through.

## What the check can and cannot say

- The page is advice, not policy. Its signs are "descriptive, not prescriptive": nearly all also occur in human text, and **one sign alone proves nothing**. The page warns that human judgment and detector tools have non-trivial error rates, and that "AI-like" writing is not automatically a problem.
- Report a **cluster level** (low / medium / high) with the matched evidence. Never write a verdict such as "this was written by AI" or "this is human".
- The word and phrase lists come from the page (English) and shift with model version and date; the page dates some patterns (for example corrective framings such as "should be treated as… rather than…" from mid-2026, and em-dash overuse only as a historical sign). **The Spanish lists below are an adaptation written for this repo, not taken from the page**; they are not calibrated. Thresholds are provisional until run against known human and AI samples.
- These files are Markdown on purpose (this repo's `events/` files, Jira links as `[ID](url)`): **ignore the Markdown-as-sign checks for `.md` files**; they only mean something in plain-text media (email body pasted without formatting, chat, Jira).
- Ineffective indicators, per the page (never count them): perfect grammar, a mix of casual and formal tone, "bland" or "robotic" prose, "fancy" prose, transition words alone ("Además", "Additionally"), unsourced content, a single em dash, a "Challenges" heading by itself.

## Invocation

`/check-ia <path> [--rewrite]` — `path` is a file. With `--rewrite`, write a new file `<name>_v2.<ext>` next to the original (see Rewrite). Without it, report only.

## Stage 0 — exact artifacts (strongest; any hit is high)

```bash
grep -n -i -E 'oaicite|contentReference|oai_citation|turn[0-9]+(search|image)[0-9]+|attributableIndex|utm_source=chatgpt|\[cite: ?[0-9]+\]|\[span_[0-9]+\]|grok_card|grok_render_citation|【[0-9]+†|attached_file:[0-9]|\[web:[0-9]+\]|ppl-ai-file-upload|:::writing|INSERT_SOURCE_URL|[0-9]{4}-XX-XX|\[Describe |\(Add your ' <file>
grep -n -i -E "i hope this helps|certainly!|of course!|would you like|let me know|espero que (esto )?(te |le )?(sirva|ayude)|¡claro!|por supuesto|¿te gustar[ií]a|no dudes en|hazme saber|házmelo saber" <file>
```

## Stage 1 — lexical density (count, then divide by words)

```bash
WORDS=$(wc -w < <file>)
# English (from the page)
grep -o -i -E "additionally|delve|crucial|pivotal|tapestry|testament|underscor|showcas|intricate|interplay|landscape|vibrant|boasts|nestled|in the heart of|groundbreaking|renowned|diverse array|serves as|stands as|a testament|reflects broader|setting the stage|evolving landscape|highlighting|fostering|valuable insights|experts argue|observers have|in connection with|associated with|while specific details|not widely documented|based on available information|it's important to note|should not be confused|should be treated as" <file> | sort | uniq -c | sort -rn
# Spanish (adaptation, not from the page)
grep -o -i -E "cabe destacar|es importante (señalar|destacar|notar)|en el panorama|panorama (actual|cambiante)|juega un papel (crucial|fundamental|clave)|rol (crucial|fundamental|clave)|testimonio de|tapiz|vibrante|rico en|sirve como|se erige como|se destaca por|a lo largo de|en el corazón de|innovador|de vanguardia|robusto|sin duda|en definitiva|en resumen|cabe mencionar|si bien los detalles|según la información disponible|no está ampliamente documentad|no debe confundirse|debe tratarse como|en conexión con|asociado con|expertos (señalan|argumentan)|diversa gama|amplia gama" <file> | sort | uniq -c | sort -rn
```

Hits per 1,000 words: 0–2 low, 3–6 medium, 7+ high (provisional). A cluster of different phrases weighs more than one phrase repeated.

## Stage 2 — structure (weak alone; skip the Markdown ones for `.md`)

```bash
# negative parallelism
grep -n -i -E "not only .* but|it'?s not .*,? (it'?s|but)|not [a-z ]+, but |rather than|no solo .* sino|no es .*,? sino|en lugar de|más que |no se trata de .* sino" <file>
# repeated stock verbs and filler (count of the most repeated word of 8+ letters)
tr -cs '[:alpha:]áéíóúñÁÉÍÓÚÑ' '\n' < <file> | awk 'length>=8' | tr 'A-Z' 'a-z' | sort | uniq -c | sort -rn | head -8
# emoji, curly quotes (curly quotes weak: Word and macOS produce them)
grep -n -P "[\x{1F300}-\x{1FAFF}\x{2600}-\x{27BF}]" <file>
grep -n -P "[\x{2018}\x{2019}\x{201C}\x{201D}]" <file>
# plain-text media only: markdown headings, thematic breaks, many bold spans
grep -c -E '^(#{1,6} |---$|\* \* \*$)' <file>; grep -o -E '\*\*[^*]+\*\*' <file> | wc -l
```

Shape numbers (print them; there is no human baseline yet, so compare versions of the same text, not texts against a threshold):
```bash
F=<file>
sed -E 's/`[^`]*`/X/g; s/https?:[^) ]+/U/g; s/\*\*//g' "$F" | grep -v '^\s*$' | tr '\n' ' ' | sed -E 's/([.!?;:])\s+/\1\n/g' | awk 'NF>0{n++; w=NF; s+=w; q+=w*w} END{m=s/n; printf "sentences=%d mean=%.1f stdev=%.1f cv=%.2f\n", n, m, sqrt(q/n-m*m), sqrt(q/n-m*m)/m}'
awk 'NF>0{t++} /^\s*([-*]|[0-9]+\.) /{l++} END{printf "list lines/nonblank: %d/%d = %.0f%%\n", l, t, 100*l/t}' "$F"
grep -o -i -E 'no probé|inferencia|asumo|según el código|no leí|no comparé|ninguno probado|ni decidido|debería' "$F" | wc -l   # hedge phrases: must be equal before and after a rewrite
```

Also read once for the cases no regex catches: rule-of-three triads, title-case headings, bold "key takeaways", inline-header lists (bullet + bold label + colon), the same sentence or idea stated twice, trailing "-ing"/"-ndo" clauses that add significance, vague attribution, and promotional or significance-inflating language.

## Stage 3 — model pass, flagged lines only

Only when Stages 0–2 give a medium or high cluster: take **just the flagged lines (and one line of context)**, not the whole file, and ask one narrow question of each: does it carry an unverifiable claim, promotional or significance-inflating wording, or filler that a human writing this ticket would not add? The page's own advice: fix the underlying problems, not the surface style.

## Output

```
Cluster: low | medium | high   (Stage 0: N hits · Stage 1: X per 1,000 words · Stage 2: signs listed)
Evidence: line N — "<excerpt>" — sign
Not counted (ineffective/weak alone): …
Not checked: …
```

Keep the evidence list short (the strongest ten at most). State what was not checked (for example, the Spanish lists are uncalibrated).

## Rewrite (`--rewrite`)

- Write `<name>_v2.<ext>` in the same folder; never overwrite the original.
- Change only what the evidence flags: repeated phrases, doubled statements, inflated or filler wording. Shorten; do not add.
- **Never change a number, a name, a URL, a code reference, a query, a hedge ("no probé", "es una inferencia mía") or a recommendation's meaning.** A hedge that was in the source stays; a claim is never made firmer.
- Keep the author's voice and language; do not "humanize" by adding mistakes or informality.
- End with a list of the changes made (original → new), so the author can reject any.

## Consistency check after a rewrite

Before handing over a `_v2`, diff it against the original and confirm it carries the same message: the set and counts of numbers, URLs and backticked terms are identical, the hedge-phrase count is equal, and no phrase became firmer (for example "veo tres caminos" must not become "hay tres caminos"). A recommendation moved to another paragraph is still the same message only if it keeps its strength ("recomiendo fuertemente").

## Calibration

The tiers are unvalidated. When an external detector is used on a text, keep the comparison (tiers, shape numbers, detector score) in that event's `data/`, never in this file. Where the detector reads high and the tiers read low, the tiers are missing a signal: do not use a "low" result to say a text reads as human.

## Language

Conversational output in English. A rewritten file keeps the language of the original (Spanish for `events/` content).
