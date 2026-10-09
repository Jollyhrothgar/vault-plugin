---
name: recall
description: >-
  Retrieve what Mike already thought about a topic — his thread notes, related
  threads, dated status snapshots, and any open reminders tied to them. Use when
  the user says "/recall", "what do I know about X", "where did I leave X", "what
  was I thinking about X", "pick up X again", "did I write anything about X",
  "when did I last send an update on X", or when re-entering a line of work after
  a task switch. Do NOT use to capture new
  thinking (use `note`) or to create reminders (use `remind`).
---

# Recall

Assemble his raw material and hand it back in his own words. You may describe
what you assembled; you may not decide what it means. **Never tell him what to
work on, what is urgent, or what matters most** — that is his judgment, and
automating it is the failure mode this whole system was designed against.

The line is **describe, don't decide.** Compressing structure and stating
checkable facts is describing. Supplying importance, urgency, or a recommendation
is deciding. And never paraphrase a thread instead of showing it — his notes are
written in his voice for his own re-entry, so your rewording is strictly worse
for the one job they have.

Vault: `~/workspace/knowledge-vault/main`

`recall` is a reader. It never writes to the vault. If a write turns out to be
needed, it goes through `note` (or `bin/vault-write begin` / `publish`) — never by
editing the `main` worktree directly.

## Output shape

**Open with a one-line header** so he knows what he is holding before he reads it:

```
mill-baseline · touched 4 days ago · 3 neighbours (retrieval, task-switching) ·
2 open reminders · 2 snapshots, latest 2026-07-29
```

Counts, names, and dates only. A header carries no content, which is exactly why
it cannot quietly become a substitute for reading the note.

**Close with a few observations** — three or four lines at most, and only ones he
could confirm by looking at the files for a second:

```
- mill-baseline is touched: 2026-07-25, but its newest snapshot is 2026-07-29.
- The open reminder "chase PROJ-5059" names a ticket this thread's Next move
  no longer mentions.
- Two of the three neighbours also carry `retrieval`.
```

Every one of those is a fact about the material, and none of them says which
matters. The test is whether he could check it by looking. "The schema decision
looks like the blocker" fails that test — it is analysis wearing an observation's
clothes. Cut it.

This is the default shape, not a ceiling or a floor. He will ask for more detail
or less; follow him.

## Inputs

- **A topic** (optional) — a slug, a title fragment, an alias, or plain words.
  Without one, list active threads.

## Steps

### 1. Resolve the query through aliases first

```bash
cat ~/workspace/knowledge-vault/main/topics.md
```

**Reads come from the `main` worktree.** Writes never touch it — every write
happens on a branch in a private worktree and lands via a squash-merged PR, so a
reader can never observe a half-written state. A reader may be slightly behind
`origin/main` if a write landed seconds ago; that costs nothing, because a read
cannot be written back on its own strength.

Run `~/workspace/knowledge-vault/main/bin/vault-write read` to fast-forward the
read worktree and print the vault path and HEAD. Call it to refresh; you do not
need to carry the HEAD anywhere.

If the query matches a topic's `aliases:` line, use the canonical `##` slug for
the rest of the search. This is what makes "switching cost" find
`task-switching`. Tell him when you resolved an alias, so the vocabulary stays
visible.

With no query: list `status: active` threads by `touched` descending, title plus
`Next move` line. That list is the entire output — no header, no observations, and
no commentary on which one he should pick up. The `Output shape` above is for
recalling a topic; here he is scanning, and a scan wants nothing on top of it.

### 2. Hop 1 — exact match

```bash
cd ~/workspace/knowledge-vault/main/threads
grep -ril "<query>" . ; ls | grep -i "<query>"
```

Search title, filename, frontmatter, and body. **Output the matching note
verbatim** — do not summarize it. He wrote it; hand it back as written.

Multiple hop-1 matches: list them and let him pick before traversing further.

#### Then list its status snapshots — dates only

Dated status snapshots live in `updates/<slug>/YYYY-MM-DD.md`, written by
`note`. List them after the thread:

```bash
d=~/workspace/knowledge-vault/main/updates/<slug>
[ -d "$d" ] && for f in "$d"/*.md; do
  printf '%s\t%s\n' "$(basename "$f" .md)" \
    "$(grep -m1 '^audience:' "$f" | cut -d' ' -f2-)"
done | sort || true
```

The `|| true` matters: no directory means no snapshots, which is the normal case,
not a failure to report.

```
Snapshots (updates/mill-baseline/):
  2026-07-22  team
  2026-07-29  team
Say a date to see one.
```

**Dates and audience only — never the contents.** The thread you just output is
living state and therefore newer than the newest snapshot, so printing that
snapshot would largely repeat what he has already read. What the series uniquely
answers is *when he last told someone something, and what he said* — a date list
answers that, and he pulls the one he wants. Then output it verbatim, like any
other note.

Chronological order is not ranking. Do not reorder by relevance, and do not
comment on which snapshot matters.

**One comparison worth making:** if the thread's `touched:` is older than the
newest snapshot's date, say so plainly —

> `mill-baseline.md` is `touched: 2026-07-25`, but there's a snapshot from
> 2026-07-29.

That means status went out to someone and never made it back into his own
thinking. It is a fact about two dates, not a judgment about what to do.

If the directory does not exist, output nothing about snapshots. Most threads
have none, and "no snapshots" is a line of noise on every recall.

### 3. Hop 2 — shared topics

```bash
grep -l "topics:.*<canonical-topic>" *.md
```

Threads carrying any `topics:` value in common with the match — **excluding the
match itself**, which this grep also returns. Show each as **title + `Where it
stands` line only**, one line each, not the full note.

### 4. Hop 3 — typed links, both directions

```bash
grep -o "\[\[.*\]\]" <match>.md          # outbound
grep -l "\[\[<match-slug>\]\]" *.md      # inbound
```

Show each as title + the link's stated reason. Inbound and outbound labelled
separately — which direction a relationship runs is information.

Present hops 2 and 3 **in file order, unranked**. Ordering by relevance is
ranking, and ranking is judgment.

### 5. Sessions — what the agents did, kept separate from what he thought

The vault holds his thinking. `session-synth` holds a different thing entirely:
automatic takeover briefings for past Claude Code and OpenCode sessions — agent
work history, written by a model about a transcript, never deposited by him and
never in his voice. **A briefing is not vault content and must never be
presented as one.** Two sources, two labels, always:

> **Your notes say** — *(the thread, verbatim, as above)*
>
> **Your sessions show** — 2 briefings mention this:
> ```
> ses_fca403dfaffeCpCaevE6N9JvWa  detector fixtures and run variance
>   Run-to-run variance is large enough that a single run is not evidence…
> c7198a26-1945-43cc-99bb-3bb948c15fd4  session-synth leverage pair
>   The fold header speaks two-way language but is emitted once per prior…
> ```
> Take one over with `takeover <id>`.

Query it with the topic you resolved in step 1:

```bash
uv run --no-project python "$(ls ~/.claude/skills/session-synth/scripts/run.py \
                                 ~/.config/opencode/skills/session-synth/scripts/run.py \
                                 2>/dev/null | head -1)" search "<topic>" --limit 5
```

The `$(ls … | head -1)` resolves whichever platform has it installed; shell
variables do not persist between tool calls, so the whole prefix goes on every
invocation. Keep `--no-project` — plain `uv run` would resolve whatever project
the caller happens to be standing in.

**If session-synth is not installed** the command fails with `can't find
'__main__' module`. Say one line — "session-synth isn't installed, so this is
vault only" — and carry on. It is a second source, not a dependency; its
absence costs a sentence, not the recall.

`search` returns id, title, and the head of the briefing — no dates and no
repo. If he wants either, `show <id>` has them; do not invent them.

**Always carry the session ids.** They are the handle for `takeover`, which is
how he re-enters the session itself instead of reading about it — the one thing
this half offers that the vault cannot.

Do not summarize the briefings, do not merge them into the thread, and do not
rank them. The rule governing the vault half governs this half unchanged:
describe, don't decide.

### 6. Open reminders

```bash
osascript -e 'tell application "Reminders" to get name of lists'
```

Then, for each list, incomplete reminders whose body mentions the thread slug:

```bash
osascript -e 'tell application "Reminders" to get {name, body, remind me date} of (reminders of list "Work - Now" whose completed is false)'
```

Filter for `thread: <slug>` in the body. Report them plainly — the obligations
half of the switch, alongside the thinking half.

### 7. Git history — only on request

Not included by default. If he asks how he got here or how his thinking moved:

```bash
cd ~/workspace/knowledge-vault/main && git log -p threads/<slug>.md
```

## Checks

- The thread note was output verbatim, not summarized.
- The header is counts, names, and dates — it carries no content.
- Every closing observation is checkable by looking, and none of them says which
  thing matters.
- Snapshots were listed as dates only, or not mentioned at all when there are none.
- Hops 2 and 3 are one line per thread, unranked.
- Alias resolution, if it happened, was stated.
- Session briefings, if any, are labelled as sessions rather than blended into
  his notes, and each one carries its id.
- Nothing in the output tells him what to work on.

## Stop Conditions

- **Nothing matches at any hop** — say so plainly and stop. Do not offer a
  guess at a related topic he did not ask for. An empty vault result means the
  thinking was never deposited, which is useful to know — but check the sessions
  half before you say it, because agents may have worked the topic hard without
  him ever writing it down.
- **The query is a single very common word** (`work`, `the`, `stuff`) — ask for
  something narrower rather than returning every thread.

## Recovery

- **Vault or `topics.md` missing**: report it and fall back to a plain grep over
  `threads/` if that directory exists. Do not create anything.
- **`osascript` permission error** (`Not authorized to send Apple events`):
  report that reminders could not be read, deliver the thread material anyway.
  Partial recall beats no recall.
- **session-synth not installed, or its database not built yet**: one line
  saying the answer is vault-only, then carry on. The vault half stands alone.
- **Failure partway through**: just stop. Reads change nothing, so there is
  nothing to release and nothing left in a bad state.
