---
name: recall
description: >-
  Retrieves what Mike already thought about a topic — his thread notes, related
  threads, dated status snapshots, his own daily logs and raw dumps, and any open
  reminders tied to them. Use when the user says "/recall", "what do I know about
  X", "where did I leave X", "what was I thinking about X", "pick up X again",
  "did I write anything about X", "what did I write this week", "show my daily
  logs", "when did I last send an update on X", or when re-entering a line of
  work after a task switch. Do NOT use to capture new thinking (use `note`) or to
  create reminders (use `remind`).
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
editing the `main` worktree directly. Mike writes in `main` himself, so
uncommitted files there are his work in progress: read them, never touch them.

Beyond `threads/` and `updates/`, the vault is Mike's space. He organizes it by
hand and moves things freely, so never assume a path inside it. Tell notes apart
by content: frontmatter `author: claude` marks an agent-written note, and label
it as agent-written whenever you show one. Every other note is his words (daily
logs, raw dumps, voice memos, raw transcripts), so the verbatim rule applies
exactly as it does to threads.

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

#### Then list his own writing that mentions it — paths and dates only

```bash
cd ~/workspace/knowledge-vault/main
grep -ril "<query>" . --include='*.md' --exclude-dir=threads --exclude-dir=updates \
  --exclude-dir=bin --exclude=AGENTS.md --exclude=README.md --exclude=topics.md | sort
```

Paths may contain spaces, so quote them. Mark any match carrying `author: claude`.

```
His writing (3):
  Coaching/Reflections/Daily Logs/2026-10-31.md
  Coaching/Voice Memos/2026-10-29-coaching-goal.md
  Coaching/Reflections/Prompts/gut-practice.md  (agent-written)
Say one to see it.
```

Dates order these; nothing ranks them. Show a file only when he asks, and then
verbatim. If nothing matches, say nothing about it.

**When he asks for his daily logs directly** ("what did I write this week",
"show my daily logs"), skip the thread hops: read `folder` from
`.obsidian/daily-notes.json` and list that folder by date, newest first, each with its first line, which is usually that day's
intention. Output any day he picks verbatim. Do not summarize a week of logs,
find themes in them, or say what they add up to unless he asks for exactly that,
and when he does, quote rather than paraphrase.

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

### 5. Open reminders

```bash
osascript -e 'tell application "Reminders" to get name of lists'
```

Then, for each list, incomplete reminders whose body mentions the thread slug:

```bash
osascript -e 'tell application "Reminders" to get {name, body, remind me date} of (reminders of list "Work - Now" whose completed is false)'
```

Filter for `thread: <slug>` in the body. Report them plainly — the obligations
half of the switch, alongside the thinking half.

### 6. Git history — only on request

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
- Nothing in the output tells him what to work on.
- His writing was listed by path and date, and shown only verbatim.
- Any agent-written reflection that was shown was labelled as agent-written.

## Stop Conditions

- **Nothing matches at any hop** — say so plainly and stop. Do not offer a
  guess at a related topic he did not ask for. An empty vault result means the
  thinking was never deposited, which is useful to know.
- **The query is a single very common word** (`work`, `the`, `stuff`) — ask for
  something narrower rather than returning every thread.

## Recovery

- **Vault or `topics.md` missing**: report it and fall back to a plain grep over
  `threads/` if that directory exists. Do not create anything.
- **`osascript` permission error** (`Not authorized to send Apple events`):
  report that reminders could not be read, deliver the thread material anyway.
  Partial recall beats no recall.
- **Failure partway through**: just stop. Reads change nothing, so there is
  nothing to release and nothing left in a bad state.
