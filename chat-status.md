# Skill: Chat Status

Cross-project skill: lists every **event** (ticket / investigation folder) worked on in the current Claude chat and the **status** of each, so the user can see what is done, what is waiting for them and what is still unverified. Read-only. Works the same across `loyalty/`, `smartpedidos/`, `operations/` and `cloud/`.

Why it exists: the user works on several tasks and projects at once, is interrupted, and priorities change. A long chat accumulates many events and the user loses track of which are done, waiting for them or unverified. By default this command answers **from the context already in the chat** (what was asked, what was written, which command outputs the user pasted or which result files were read), with **no tool calls**. An optional `verify` argument checks the event folders.

## Invocation

`/chat-status [verify] [event or window]`

- **No argument (default): context only.** Build the table from the conversation. **Do not run any tool**: no `find`, no file reads. This is the fast answer the user normally wants.
- `verify`: after building the table from context, check it against the event folders (the steps under "Verification mode" below) and report any disagreement. Only then read files; invoking `verify` counts as the explicit request to read `events/`, which is otherwise write-only by project convention.
- An optional event name or ticket ID limits the answer to that event.

## Sources (default mode)

1. **The current conversation**, including any compaction summary at its start: the events named, the requests, the scripts written, the commands handed over, the outputs the user said they ran, the files read in this session and what they showed, and the drafts written.
2. If the context was compacted and an event's state is unclear, say "unknown from context" for that field; do not guess. (Only in `verify` mode, or if the user asks, read the session transcript `~/.claude/projects/<project path with dashes>/<session id>.jsonl` or the event folders.)

Evidence rules even in context-only mode: a command is **run** only if the chat shows its output (a log read, a pasted result) or the user said they ran it **and** the result was seen; "handed over" or "pasted back" alone is not a run. Mark anything the user only stated as **user-stated**.

## Steps (default mode)

1. List the events in the chat in order of first appearance; group sub-tasks under their parent ticket.
2. For each, set one status from the vocabulary, with the evidence you can name from the chat (log file names read, results seen, drafts written).
3. List what is pending for each, and what was handed to the user and not answered.
4. Mark what is **user-stated** and what is **unverified** (never flatten a hedge).
5. Output as below. No tool calls.

## Verification mode (only with `verify`)

1. **Find the events** (read-only):
   ```bash
   cd /home/dpaniagua/Documentos/git/bots
   find loyalty/events smartpedidos/events operations/events cloud/events -mindepth 1 -maxdepth 1 -type d -newermt "<window start>" 2>/dev/null | sort
   find loyalty/events smartpedidos/events operations/events cloud/events -type f -newermt "<window start>" -printf '%TY-%Tm-%Td %TH:%TM %p\n' 2>/dev/null | sort | tail -200
   ```
   An event is **in scope** when any file inside changed in the window, or the conversation names it. Group changed files by event folder.
2. **For each event, read:** the first lines and `Status:` of `investigation.md`; the title, ticket and section headings of `ops.md`; the names and timestamps in `scripts/` and `data/`.
3. **Derive what actually happened** from the files, to compare with the table built from context:
   - a script exists but no `dry`/`apply` log → **written, not run**;
   - a `*-dry-log-*` without an `*-apply-log-*` → **dry run only**;
   - an `*-apply-log-*` → read its last lines: did it end with the rollback line (applied) or with a stop/`AVISO`/`RECHAZADO`/error message (stopped or partial)?
   - an `ops.md` containing only a "Qué pasó" section → **ticket not written yet**;
   - a read-back after an apply (`after-*.json`, "despues" lines) → **applied and read back**; none → **applied, not read back**.
4. **Assign one status** per event from the vocabulary below, and list the evidence file for it.
5. **Pull the open items** from the `Open` / `Next steps` / pending sections of `investigation.md` and from `Pendientes` in any email draft, without inventing new ones.
6. **Cross-check the user's statements** (done / confirmed / decided) against the files; mark each one **verified** (a file shows it) or **user-stated** (only said in the chat).

## Status vocabulary

| Status | Meaning |
|---|---|
| Not started | folder exists, no investigation content yet |
| Investigating | read-only checks done or running; no change proposed |
| Waiting for user | a command or decision was handed over and no result is in `data/` yet |
| Ready to apply | dry run is clean, apply not run |
| Applied, not verified | apply ran; no read-back, no real event seen |
| Applied and verified | apply ran and a read-back or a real event confirms it |
| Blocked | a stop message, a rejected step or an external dependency prevents progress |
| Partial | some steps applied, others stopped or pending |
| Done | the request is satisfied and nothing is pending except housekeeping |
| Drafted | a document or email exists and has not been sent/published (the chat cannot see sending) |

## Output (English, in the chat; do not write files)

**Table format only.** Every part of the answer is a table. No bullet lists, no numbered lists and no explanatory paragraphs under the tables; whatever needs explaining goes in a table cell (keep cells short). The only text outside a table is the one-line header and the one-line next step.

1. A one-line header: window, number of events found.
2. **Events table**, one row per event, sorted with the ones that need the user first:

   | Event | Status | Done / verified | Unverified | Next |
   |---|---|---|---|---|

   **Terminal width rule (the user's terminal turns wide tables into stacked blocks and cuts cells):** at most 5 columns, each cell at most about 45 characters, one short phrase per cell, no file names in the table (put the evidence in the conversation if asked), the Jira ID once in the header line, not per row. If a row needs more, shorten it; never add columns.
   - **Done / verified:** what was changed and read back; "-" if nothing.
   - **Unverified:** hypotheses, things only the user said, anything not checked; "-" if nothing.
3. **Needs you now** table, in the order they matter: commands or decisions handed over and not yet answered.

   | Order | Item | Why it matters |
   |---|---|---|

4. **Disagreements** table, only when a user-stated item is not shown by the files or the files contradict the chat; omit the table when there are none and say "Disagreements: none found" in the next-step line.

   | Item | User-stated | What the files show |
   |---|---|---|

5. End with **one concrete next step** in a single line.

## Rules

- **Default mode makes no tool calls.** In `verify` mode it is read-only: do not run `git add/commit/push`, any cloud CLI, Zabbix/Graylog API calls, or database commands. Reading local files and `find`/`ls`/`grep`/`cat` are fine.
- **No names** of people in the output when they come from files; use roles. Account/user ids are fine.
- **Never flatten a hedge.** If a file says "not verified", "hypothesis" or "proposed", the status line says so; do not turn it into "done".
- **Do not infer that a command ran** because it was handed over or pasted back: only a file in `data/` shows that it ran (a pasted command is not a run).
- Do not list events only because they are old; stay inside the window plus conversation-named events.
- If an event folder has no `investigation.md`, say so; do not create one.
- Do not edit `investigation.md`, `ops.md` or any other file as part of this command.
