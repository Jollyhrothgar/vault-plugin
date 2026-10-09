---
name: note
description: >-
  Interview Mike about what he is thinking, then fold it into a thread note in
  his vault so he can re-enter that line of work later. Use when the user says
  "/note", "let's capture this", "I'm switching off of X", "park this",
  "write this up", "brain dump", or is about to drop a line of work and would
  lose the context. Also use when they describe thinking that has no thread yet,
  when an agent that just did the work should fold what it learned into a thread,
  or when they ask for a dated status snapshot ("where are things at on X",
  "write a status update").
  Do NOT use for obligations or follow-ups (use `remind`) or for retrieving
  existing notes (use `recall`).
---

# Note

Interview, then fold. The interview is the point — Mike deposits context he
would otherwise lose on a task switch. Never reconstruct what he was thinking
from files or git history; ask him.

Vault: `~/workspace/knowledge-vault/main`
Threads: `threads/<slug>.md` · Vocabulary: `topics.md`

## Inputs

- **A topic** (optional) — `/note search relevance` targets that thread. Without
  one, ask what he is switching off of.
- **Whatever he already said** — if the conversation already covered where
  things stand or who is involved, do not re-ask it.

## Steps

### 1. Open a write and read the vault

Take a private write worktree first, then read the vault *inside it*. Everything
you read and everything you write lives in that one worktree, so the interview
and the fold are guaranteed to be based on the same snapshot.

```bash
eval "$(~/workspace/knowledge-vault/main/bin/vault-write begin | sed 's/^/VW_/')"
printf 'writing in %s (token %s)\n' "$VW_vault" "$VW_token"
ls "$VW_vault/threads/"
cat "$VW_vault/topics.md"
for f in "$VW_vault"/threads/*.md; do echo "--- $f"; sed -n '1,12p' "$f"; done
```

`begin` cuts the worktree from a freshly fetched `origin/main`. It takes no lock,
so it never blocks and never blocks anyone else — another session can be writing
a different thread at the same time and both will land. You need every thread
title, its `Where it stands` line, and the full topic vocabulary before
interviewing. A hundred threads is a trivial read; read them all rather than
grepping for a guess.

**Keep `$VW_token` and `$VW_vault` for the rest of the run.** The token publishes
the fold in step 8; the path is the only place you may write. Never read or write
`~/workspace/knowledge-vault/main` directly — that is the shared read worktree.

If you finish the interview and Mike decides there is nothing to fold, release
the worktree with `vault-write abort "$VW_token"`. It refuses if you are holding
unpublished content, which is the intended safety net.

### 2. Interview — one question at a time

Ask **one question per message**, roughly five total. Adapt: skip anything he
has already answered, and follow a thread of his if it is more interesting than
your next scripted question.

1. What were you working on / what's top of mind?
2. Where did you leave it?
3. What are you thinking, and what are you unsure about?
4. Who's involved — anyone waiting on you, or you on them?
5. What's the next move when you come back?

Do not batch these into one message. Do not summarize his answers back at him
between questions — that is the spoon-feed tax, and it wastes the interview.

**When the thread already has a `Next move`, question 2 becomes that question:**

> Last time the next move was "validate the tight-loop agent's PR and merge it."
> Did that happen?

Ask it verbatim from the note, not paraphrased. This replaces "where did you
leave it" rather than adding to the interview — it gets better state, and it is
the only point in the system where a note gets graded. If the answer is that he
could not tell what it meant, or had to go dig up which PR, that is the last
fold's referents failing out loud; carry it into step 7 and anchor them this
time.

Answering question 1 is what identifies the thread, so **match it (step 3) and
read it in full before asking question 2** — the survey in step 1 stops above
`Next move`. Never overwrite a `Next move` without asking about it first.

#### When you were the working session

Sometimes the agent running `note` *is* the session being noted — it just spent
an hour restarting the workspace, landing the patch, launching the run. Then the
facts of that session are first-hand, not reconstructed, and re-asking them
wastes the interview.

In that mode only:

- **Fold session facts directly.** What was done, what broke, what state things
  are in — you were there. Do not make him narrate it back to you.
- **The interview shrinks to what you cannot know.** Judgment only: priorities
  and reprioritizations, the north-star target, what he is unsure about, who is
  waiting. Two or three questions, not five.
- **The interview can run the other way.** He may be the one interviewing you —
  "what did the batch actually score?", "did the schema change land?" Answer
  from the session, plainly. His questions and your answers are both fold
  material.
- **Contradiction-surfacing (step 4) and show-before-write (step 7) stay
  mandatory.** They are the entire safeguard here. A session-grounded fold
  carries more agent-authored text than any other, so it needs the check more,
  not less. On the 2026-07-29 `mill-baseline` fold the thread's `Next move` said
  "drive PROJ-5059 myself" while his new statements demoted that ticket; naming
  both readings and asking him is what made the fold correct.

This mode requires that you *were* the session. **Being able to read the files is
not the same thing.** On a cold invocation — fresh context, or work someone else
did — you know nothing first-hand and the general rule holds: ask him, never
reconstruct his thinking from files or git history.

### 3. Match or create the thread

Match after question 1, before the rest of the interview. Propose the matching
thread by name, or propose a new kebab-case slug. Never silently create a thread
whose title is a near-duplicate of an existing one — if it is close, ask whether
this is the same line of work.

**Read the matched thread in full before continuing.** Step 1 read only its
first 12 lines; the contradiction check (step 4) and the grading question
(step 2) both need the whole file.

Read it from your write worktree — the same snapshot step 1 opened, so it cannot
have moved:

```bash
cat "$VW_vault/threads/<slug>.md"
```

### 4. Surface contradictions — do not skip this

If what he just said conflicts with what the thread note already says, **stop
and name both readings before folding**:

> The note says keywords should boost, not filter. You just said the filter is
> fine and the embedding is the problem. Which is it?

This is the highest-value behavior in the skill. A wrong note is worse than no
note, and this is the mechanism that engages his thinking rather than filing on
his behalf. Never resolve a contradiction yourself by picking the newer
statement.

### 5. Assign topics from the vocabulary

Propose `topics:` values that already exist in `topics.md`, **checking aliases
too** — a topic whose alias matches what he said is a hit, not a miss.

Coin a new topic only when no existing one fits. Coining requires writing its
full entry in the same fold:

```markdown
## <slug>
<definition>
Not: <what it excludes>
aliases: <synonyms he actually used>
```

Never write a bare `topics:` value that has no entry in `topics.md`.

### 6. Propose typed links, both directions

From the thread titles read in step 1, propose `Related` entries with a stated
reason. Write **both sides** — if this thread links to `task-switching`, add the
reciprocal entry to `task-switching.md` in the same fold. Backlinks are never
left missing.

Shared `topics:` already give co-occurrence for free. Only propose a `Related`
link for a *typed* relationship a shared topic cannot express — "supersedes",
"is why I care about", "blocked by".

### 7. Show the fold, then write

Show the diff (or the full new file) and get approval before writing. The fold
updates:

- `touched` to today
- any new `topics` and `people`
- the four body sections
- the reciprocal side of any new link
- `topics.md`, if a topic was coined

After approval, write the files into `$VW_vault` — the worktree from step 1, and
the only place you may write. Nothing is published until step 8, so an abandoned
run leaves the vault untouched.

Thread note format:

```markdown
---
status: active          # active | parked | done
topics: [knowledge-management, task-switching]
people: [jane-doe]
touched: 2026-07-28
---
# Search relevance

## Where it stands
Blind queries return junk — the KM query ("knowledge
management tooling") filtered 88 of 94 candidates below threshold. Boost-only
patch drafted on branch `search-boost`, no PR yet.
[PROJ-1234](https://linear.app/<workspace>/issue/PROJ-1234) tracks it.

## What I'm thinking
Project keywords should boost, not filter. Unsure whether that's the real
cause or whether the embedding is wrong — a boost-only patch tells us which.

## Next move
Re-run the KM query against those 88 entries with `search-boost` checked
out; compare recall@10 to the baseline numbers in
[PROJ-1234](https://linear.app/<workspace>/issue/PROJ-1234).

## Someday
- Try a learned reranker instead of keyword boosting — only if aliases stop being enough.

## Related
[[task-switching]] — why recall latency matters at all
```

Five sections, in this order. `Where it stands` is state. `What I'm thinking` is
open thinking and uncertainty, including where it is heading. `Next move` is the
re-entry point. `Someday` is parked ideas for this thread — **not commitments**,
and the destination for drained Someday captures (step 9). `Related` is
wiki-links with a reason.

`Someday` is the only optional section; omit it when empty rather than leaving a
placeholder. The other four always appear.

**The note must stand alone.** It will be read by someone holding none of your
working memory — Mike in three weeks, or an agent that just opened the file
cold. The example above is the standard: every reference in it is a link, an
identifier, a branch, or a command.

**Hyperlink everything linkable.** Any reference with a canonical URL is written
as a markdown link, never bare: Linear tickets
(`[PROJ-1234](https://linear.app/<workspace>/issue/PROJ-1234)`), GitHub PRs
(`[#57](https://github.com/<org>/<repo>/pull/57)`), documents, dashboards. A
bare `PROJ-1234` or `#57` in a fold is a defect — the note is a re-entry point,
and a link Mike can click beats an ID he has to go look up.

#### Status snapshots go to `updates/`

A thread is living state, rewritten in place, and a thread carrying an
audience-facing status summary stops being his thinking. So status lives beside
it, in the vault, as a dated file that is *allowed* to go stale:

```
updates/<thread-slug>/YYYY-MM-DD.md
```

This is the successor to the daily log, and one rule keeps it from failing the
same way. Both earlier attempts at this system died of unread accumulation: a
per-day log nobody re-read, then a knowledge graph nobody read.

**A named audience is the precondition. No audience, no update.** Write one only
when there is someone it is for — a team, stakeholder, person, or channel — and
name them in `audience:`. If he cannot say who it is for, the thinking belongs in
the thread instead. `audience: me` is not valid; a snapshot addressed to himself
is the daily log returning, and the thread already serves that purpose.

```markdown
---
thread: mill-baseline
date: 2026-07-29
audience: team          # team | <person> | <channel>; required, never "me"
---
# Mill baseline — status 2026-07-29

Snapshot, not living state. Current thinking: [[mill-baseline]].

## Where things are
...

## What changed since the last update
...

## What's next
...
```

Three rules keep the series worth having:

- **Never edit a written update.** It is a point in time. A wrong one is
  superseded by the next dated file, not corrected in place.
- **It may be agent-authored.** This is the one artifact in the vault that may
  be — a dated snapshot of a session you ran is a report, and reports are not a
  substitute for his thinking. Threads are still his.
- **The thread does not link down to it.** `updates/<slug>/` is derivable
  from the slug, so a link would mean editing the thread on every update for
  nothing. Updates link *up* to the thread, never the reverse.

A worktree's `.local/` is still the right home for ephemeral artifacts — logs,
scratch scripts, benchmark output — that should die when the worktree does. Link
to those from an update if useful, knowing the link will rot. Anything meant to
outlive the worktree belongs in the vault.

**Anchor what has no URL.** Commit SHAs, branch names, run IDs, and exact
commands with their flags are cheap to write now and expensive to recover later.
"Re-run at a pinned SHA" records nothing; "re-run at `api-server` `abc1234`" records the
decision. When something is genuinely unknown, write it down as unknown —
"owner unknown, find who owns api-server" is self-contained, while "push it to
whoever owns api-server" disguises an open question as an instruction.

### 8. Publish

**One command. Name your token and every path you touched:**

```bash
~/workspace/knowledge-vault/main/bin/vault-write publish "$VW_token" \
  -m "vault: <thread title> — <what changed>" \
  -- threads/<slug>.md topics.md
#    + reciprocal-link files, + updates/<slug>/<date>.md
```

Paths are relative to the vault root. `publish` stages exactly those paths,
commits, pushes the branch, opens a PR, and squash-merges it — then removes the
worktree and fast-forwards the shared read worktree. There is no sequence to get
right and nothing to clean up by hand.

Two outcomes are normal and neither is yours to work around:

- **"changed but was not listed"** — a path is dirty that you did not name. In
  your own private worktree that means you forgot it, so add it. Never widen the
  list with a wildcard to make the error go away.
- **"topics not in topics.md"** — you used a topic the vocabulary does not
  define. The write proceeds anyway, because a note is worth more than a
  vocabulary entry, but say so in your summary so the drift is visible.

One outcome stops for a human, by design:

- **Exit 3, "CONFLICT"** — another write landed on the same lines while you were
  interviewing. The PR stays open and your worktree stays intact; nothing is
  lost. **Do not resolve it yourself.** A conflict means two writers rewrote the
  same thinking, and only Mike can say which reading is right. Report the PR URL
  and the worktree path, say which thread and which section collided, and stop.

Mike runs **parallel agent sessions**, which is why `publish` takes exact paths
and refuses anything else. Do not fall back to `git add` and `git commit` by
hand, and never edit `~/workspace/knowledge-vault/main` — that is the shared read
worktree, and writing there is the one thing the design cannot protect against.

Publish every fold. Prune-by-delete depends on git holding the history, so an
unpublished vault is an unsafe vault.

If `publish` fails *after* opening the PR, the commit is already safe on the
branch. Run `vault-write merge "$VW_token"` to finish the job rather than
starting over. `vault-write status` shows every write in flight.

### 9. Drain the Someday inbox

`Work - Someday` in Reminders.app is a **capture inbox, not storage** — it exists
so Mike can drop an idea from his phone when away from a computer. It only stays
honest if it gets emptied, so check it on every run:

```bash
osascript -e 'tell application "Reminders" to get {id, name, body} of (reminders of list "Work - Someday" whose completed is false)'
```

If it is empty, say nothing and move on. For each item that is there:

1. Propose the thread it belongs to, from the titles read in step 1. If none fits,
   say so — that is a signal to start a thread, not a reason to invent a home.
2. Grep the committed target thread for `<!-- reminder-id: <id> -->`. If it is
   already present, do not fold again; mark only that reminder ID complete and
   continue. Grep the **committed** file (`git show HEAD:<path>`), not the working
   copy — an uncommitted marker from another writer is not durable evidence.
3. Fold the approved items into their threads' `## Someday` sections as bullets,
   in Mike's words, ending each bullet with `<!-- reminder-id: <id> -->`. Create
   the section if a thread lacks one.
4. Publish with its **own** `vault-write begin` / `publish` pair, separate from
   the fold in step 8 — a drain that fails must not take the thread fold with
   it. Open it after step 8 has merged, so the drain branches from a vault that
   already contains your fold.
5. **Mark each reminder complete only after that fold succeeds:**

```bash
osascript -e 'tell application "Reminders" to set completed of (first reminder of list "Work - Someday" whose id is "<exact id>") to true'
```

Complete, never delete — the capture stays recoverable in Reminders' completed
items if the fold turns out wrong. Completing also removes it from the
`completed is false` query, so the drain is idempotent and re-running is safe.

**Never mark an item complete without writing it somewhere first.** That silently
destroys a capture, which is worse than an untidy inbox.

If draining would touch more than ~3 threads, confirm the scope with Mike before
writing — the same limit as any other fold.

### 10. Offer consolidation, only if warranted

If the threads you touched show one of these, propose it — once, briefly:

- **Merge:** near-duplicate titles, or two threads that link only to each other.
- **Split:** a `What I'm thinking` carrying three unrelated concerns.

He decides. Do not turn this into a review of the whole vault; that is `prune`.

## Checks

- Questions were asked one at a time, not batched.
- Session facts were folded first-hand rather than re-asked **only** because you
  ran that session — not because you could read the files.
- The previous `Next move` was asked about, not silently overwritten.
- Every `topics:` value exists in `topics.md`.
- Any new topic has a definition and a `Not:` line.
- Every new `Related` link has its reciprocal written.
- Every Linear ticket, PR, or document reference is a hyperlink, not a bare ID.
- The `Next move` is executable by someone holding only this note.
- The fold was shown before it was written.
- Any status summary went to `updates/<slug>/<date>.md`, dated, marked as
  a snapshot, and named a real audience — not into the thread body and never
  written without someone to write it for.
- The commit landed and was pushed.
- The vault-writer lock was released by a successful publish.
- No Someday reminder was marked complete before its fold was committed.
- Every drained Someday item carries its stable reminder ID, and only that ID was
  marked complete.

## Stop Conditions

- **A contradiction he has not resolved** — stop and ask. Never fold both
  readings in, and never pick one for him.
- **A `Next move` you could not execute yourself** — read it back as an agent
  holding only this file. If you cannot act on it without excavating referents,
  name what is missing and ask him for it. If he does not know either, write it
  down as unknown. Never fold the vague version and move on.
- **He gives one-word answers to the first two questions** — he does not want
  to be interviewed right now. Offer to capture just the `Next move` line and
  stop.
- **No thread matches and he does not want a new one** — capture nothing;
  suggest `remind` if it is really an obligation.
- **More than ~3 threads would need edits** — confirm the scope before writing.
- **The vault-writer lock is held** — report its owner metadata and stop. Never
  remove another session's lock to make progress.

## Recovery

- **Vault directory missing**: do not create it silently — tell him the vault
  path is absent and stop. A vault created in the wrong place fragments the
  graph.
- **`topics.md` unparseable** (a `##` entry with no definition): report the bad
  entry and fold the thread anyway using existing topics only; do not coin a new
  topic on top of a broken vocabulary.
- **Your write worktree is dirty when you arrive at publish**: that is your own
  work, because `begin` cuts a fresh worktree nobody else can see. List every
  path you touched. `publish` rejects any dirty path you did not list, which is
  the behavior working, not a problem to route around.
- **`publish` exits 3 with "CONFLICT"**: another writer changed the same lines.
  Nothing is lost — the PR is open and the worktree intact. **Stop and hand it
  to Mike**; only he can say which reading of the same thinking is right. Never
  resolve a vault conflict yourself and never force-push.
- **`publish` failed after opening the PR**: the commit is durable on the branch
  and the PR exists. Run `vault-write merge "$VW_token"` to finish. If the merge
  is refused, report the PR URL and stop.
- **A stale write worktree is lying around**: harmless. It holds a branch nobody
  else can see and blocks no one. `vault-write status` lists what is in flight;
  `vault-write abort <token>` clears one that holds nothing unpublished, and
  refuses if it holds content — which is the guard telling you a real partial
  write is there. Report that; never discard it.
