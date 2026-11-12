# note: drain the Someday inbox (dormant)

Removed from `skills/note/SKILL.md` on 2026-11-11. It was step 9, between
"8. Publish" and "Offer consolidation" (which became step 9). Restore it there
and renumber.

## Step text

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


## Lines that were in `## Checks`

- No Someday reminder was marked complete before its fold was committed.
- Every drained Someday item carries its stable reminder ID, and only that ID was
  marked complete.
