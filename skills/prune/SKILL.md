---
name: prune
description: >-
  Sweeps Mike's vault for rot — stale threads and a drifting topic vocabulary —
  and acts on his decisions. Use when the user says "/prune", "the vault feels
  noisy", "clean up my notes", "too many threads", "my topics are a mess", or
  asks to consolidate or tidy the vault. Do NOT run this on a schedule or
  unprompted; it only runs when he asks. Do NOT use to capture (use `note`) or
  retrieve (use `recall`).
---

# Prune

Two passes, threads then vocabulary — pruning threads changes which topics are
singletons, so the order matters.

**He decides, you propose and execute.** Every delete, merge, and retirement is
his call. Your job is the mechanical rewrite afterward.

Vault: `~/workspace/knowledge-vault/main`

**Scope is `threads/`, `updates/`, and `topics.md` only.** Never propose deleting,
merging, or rewriting anything under `dumps/` or `reflections/`, and above all
`reflections/daily-logs/`, his daily writing. Old logs are not rot; an unread
log is still his record of the day. Never touch uncommitted files in `main`;
those are his work in progress.

**Never run this from a schedule, a cron, or a hook.** Pruning on a timer is the
tool deciding the vault needs attention; pruning when he notices noise is him
deciding. Same operation, opposite direction — and the scheduled version is what
killed the previous attempt at this system.

## Pass 1 — threads

### 1. List by staleness

```bash
eval "$(~/workspace/knowledge-vault/main/bin/vault-write begin | sed 's/^/VW_/')"
printf 'writing in %s (token %s)\n' "$VW_vault" "$VW_token"
cd "$VW_vault/threads"
for f in *.md; do
  printf '%s\t%s\t%s\t%s\t%s\n' \
    "$(grep -m1 '^touched:' "$f" | cut -d' ' -f2)" \
    "$(grep -m1 '^status:' "$f" | cut -d' ' -f2)" \
    "$(grep -m1 '^# ' "$f" | sed 's/^# //')" \
    "$(sed -n '/^## Next move$/ {n;p;q;}' "$f")" "$f"
done | sort
```

Reading takes no lock. Present oldest-`touched`
first: title, status, and the `Next move` line. The `Next move` is what tells him
whether a thread is still alive. Record the HEAD printed above; the decisions
below apply to that snapshot.

### 2. Take a decision per thread

For each: **keep**, **delete**, or **merge into another thread**. Ask in
batches he can actually answer — a handful at a time, not one message per
thread and not forty at once.

### 3. Execute

You are working in the private worktree opened in step 1, so nothing can change
underneath you while he is deciding. Publish it with `$VW_token` in step 4.

Run every command in this pass from your own write worktree — never from the
shared read worktree at `~/workspace/knowledge-vault/main`:

```bash
cd "$VW_vault"
```

**Delete is `git rm` — and it takes the thread's status snapshots with it:**

```bash
git rm threads/<slug>.md
git rm -r updates/<slug>/     # only if it exists; most threads have none
```

`updates/<slug>/` holds dated snapshots written by `note`, each linking
back to `[[<slug>]]`. Leaving the directory behind orphans every one of them
against a note that no longer exists — the same silent break as a dangling
wiki-link, just pointing the other way.

No archive folder, no `status: archived`, no tombstone. This holds for snapshots
too: they are never edited after they are written, so git already preserves them
exactly. `git log --diff-filter=D -- threads/ updates/` recovers
anything regretted. Never consolidate or summarize old snapshots — each one is a
frozen record of what was true that day, including what he later turned out to be
wrong about, and a summary sands that off.

**Merge** means: fold the loser's content into the survivor (show him the merged
body first), rewrite any `[[loser]]` links across all threads to point at the
survivor, then `git rm` the loser — including `git rm -r
updates/<loser>/`. The loser's snapshots die with it rather than moving
under the survivor: a moved snapshot's `thread:` frontmatter would name a dead
slug, and same-day dates would collide. Git still holds them.

```bash
grep -rl "\[\[<loser-slug>\]\]" .    # find inbound links before removing
```

Leaving a dangling wiki-link is a silent break — always rewrite inbound links
before deleting the target.

### 4. Publish pass 1 on its own

```bash
cd "$VW_vault"
git status --short                # confirm only your own paths are dirty
~/workspace/knowledge-vault/main/bin/vault-write publish "$VW_token" \
  -m "vault: prune threads — <n> deleted, <n> merged" \
  -- <exact changed and deleted paths, relative to the vault root>
```

List deletions too — `git rm` leaves them dirty, and `publish` refuses any dirty
path you did not name. If it exits 3 with a conflict, another writer changed the
same lines while he was deciding: **stop and hand it to Mike.** Never resolve a
vault conflict yourself.

A separate transaction from pass 2, so a bad vocabulary rewrite reverts without
losing the thread pruning. Pass 2 opens its own `begin`.

## Pass 2 — vocabulary

### 5. Compute the rot metrics

All by grep, on demand. Never write these to a file.

```bash
eval "$(~/workspace/knowledge-vault/main/bin/vault-write begin | sed 's/^/VW_/')"
printf 'writing in %s (token %s)\n' "$VW_vault" "$VW_token"
cd "$VW_vault"

# topics declared in the vocabulary
grep '^## ' topics.md | sed 's/^## //' | sort > /tmp/declared

# topics actually used, with counts — singletons are the leading indicator
grep -h '^topics:' threads/*.md | sed 's/^topics: *\[//; s/\]//' | \
  tr ',' '\n' | tr -d ' ' | grep -v '^$' | sort | uniq -c | sort -n

# used but never declared
grep -h '^topics:' threads/*.md | sed 's/^topics: *\[//; s/\]//' | \
  tr ',' '\n' | tr -d ' ' | grep -v '^$' | sort -u | comm -23 - /tmp/declared

# threads carrying no topics at all
grep -L '^topics:' threads/*.md

# threads with no Related link in either direction
for f in threads/*.md; do
  s=$(basename "$f" .md)
  grep -q '\[\[' "$f" || grep -rql "\[\[$s\]\]" threads/ || echo "orphan: $s"
done
```

Report, in this order:

1. **Topics used by exactly one thread** — the leading indicator of vocabulary
   rot. A vault of singletons has no co-occurrence graph at all.
2. Topics used but absent from `topics.md`, or declared with no definition.
3. Threads with no `topics:`.
4. Threads with no links in or out.

### 6. Take a decision per problem topic

For each singleton or undeclared topic: **merge into an existing topic**,
**keep** (some topics are legitimately new), or **retire**.

Propose the merge target yourself — near-duplicate slugs and overlapping
definitions are mechanical to spot. He picks the survivor.

### 7. Execute the vocabulary rewrite

This is a second, independently published pass. Re-read the vault HEAD after
pass 1 published — it advanced — and use that value in step 8. If it differs from
the snapshot used to compute the rot metrics, recompute them before writing.

For a merge:

1. Rewrite `topics:` across every affected thread, replacing the loser with the
   survivor. De-duplicate if a thread already carried both.
2. Demote the loser to an entry on the survivor's `aliases:` line — the old term
   stays findable by `recall`, which is the whole point of aliases.
3. Remove the loser's `##` entry from `topics.md`.

For a retirement with no survivor: remove the `##` entry and strip the value
from any thread frontmatter.

Show the full set of edits before writing. This touches many files at once, so
confirm the repo is clean first — `git status` — and commit before starting so
one `git revert` undoes the whole rewrite.

### 8. Publish pass 2

```bash
cd "$VW_vault"
git status --short                # confirm only your own paths are dirty
~/workspace/knowledge-vault/main/bin/vault-write publish "$VW_token" \
  -m "vault: prune vocabulary — <n> topics merged, <n> retired" \
  -- <exact changed paths, relative to the vault root>
```

## Checks

- Pass 1 committed before pass 2 began.
- No `[[link]]` points at a deleted thread.
- No `updates/<slug>/` directory outlives the thread it was written for.
- Every `topics:` value still resolves to a `##` entry in `topics.md`.
- Every merged-away topic survives as an alias.
- Rot metrics were reported, not written to a file.
- Nothing under `dumps/` or `reflections/` was proposed for deletion or changed.
- The vault-writer lock was released after each committed pass.

## Stop Conditions

- **Fewer than ~12 threads** — say so and stop. The rot metrics are noise at
  that size; everything looks like a singleton in a young vault.
- **He wants to delete a thread with inbound links** — name the inbound threads
  and confirm before removing, since those links will break.
- **A vocabulary rewrite would touch more than ~15 threads** — confirm the full
  scope before writing, and offer to split it across two commits.
- **Your write worktree is dirty with unrelated work** — stop. Rewrites across
  many files need a clean baseline to be revertible. A dirty `main` is normal,
  since Mike writes there, and is not a reason to stop.
- **Another write is in flight** — `vault-write status` lists them. This is not
  a reason to stop: writers no longer block each other, and a conflict, if one
  comes, surfaces at publish time.

## Recovery

- **`publish` exits 3 with a conflict**: another writer changed the same lines.
  Nothing is lost — the PR stays open and your worktree intact. **Stop and hand
  it to Mike**; only he can say which reading is right. Never resolve it
  yourself and never force-push. Mid-prune this is safe — pass 1 is already
  merged.
- **Mid-rewrite failure**: restore the exact paths changed in this pass if
  uncommitted, then run `vault-write abort "$VW_token"`. It refuses while you
  hold unpublished content, which is the guard working. If the pass is already
  committed on its branch, `git revert <sha>` in the worktree and publish that
  revert. Never hand-repair a half-finished topic rewrite; revert and restart.
- **Dangling links found after a delete**: rewrite them to the survivor, or
  remove the `Related` bullet entirely — a link with a reason but no target is
  worse than no link.
- **`comm` or `uniq` pipeline errors** on an empty vault: report that there is
  nothing to measure yet and stop.
- **Publish rejected**: leave the commits on the branch, report the PR URL and
  the token, and do not force-push. `vault-write merge <token>` finishes a write
  whose PR is already open.
