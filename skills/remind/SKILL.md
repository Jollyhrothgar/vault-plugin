---
name: remind
description: >-
  Triages a reminder request into the right macOS Reminders.app list, links it to
  the vault thread it belongs to, and creates it there. Work items land in a work
  list bucketed by urgency (Work - Now, Work - Soon, or Work - Someday as a
  phone-capture inbox that `note` later drains); non-work items route to a matching
  personal list (Groceries, family lists) or the default Reminders list. Use when
  the user says "/remind", "remind me to", "add a reminder", "make a note in my
  reminders", "follow up on X tomorrow", or asks to track a follow-up outside the
  current session. Do NOT use for Linear issues, calendar events, or scheduled
  agent runs, and do NOT use to capture thinking (that is `note`).
---

# Remind

Create a reminder in macOS Reminders.app, choosing the list by triaging the
request into work vs. personal, and work items by urgency. Link it to the vault
thread it belongs to so `recall` can find it later.

A reminder is the **obligations** half of a task switch. The **thinking** half
is `note`. If the request is really "help me not lose what I was thinking," use
`note` instead.

## Inputs

- **The reminder text** — what to be reminded of. Required; ask if the request
  is only "remind me".
- **A trigger time** (optional) — "tomorrow at 10am", "Friday", "in 2 hours".
  If absent, create the reminder without a due date.
- **Context** (optional) — links, PR numbers, ticket IDs from the conversation
  worth preserving. Put these in the reminder `body`, not the title.

## Steps

### 1. Read the available lists

List names are user-editable and drift over time — never assume the list
you want exists. Query first:

```bash
osascript -e 'tell application "Reminders" to get name of lists'
```

### 2. Triage into a list

Apply the first matching rule:

| Signal in the request | Target list |
| --- | --- |
| Job-related: PRs, reviews, tickets, deploys, meetings, teammates | A `Work - *` list, by urgency (below) |
| Shopping/food items | `Groceries` |
| A person's name matching a personal list (e.g. `Julian`) | That person's list |
| Everything else | `Reminders` (default) |

Work urgency buckets:

- **`Work - Now`** — has a deadline within ~2 days, blocks someone, or the
  user says "today", "tomorrow", "urgent", "before standup".
- **`Work - Soon`** — this week / next week, dated follow-ups without
  same-day pressure.

- **`Work - Someday`** — "eventually", "at some point", ideas, no date and no
  urgency signal. **This is a capture inbox, not storage.** It exists so Mike can
  drop an idea from his phone when away from a computer; `note` drains it into the
  owning thread's `## Someday` section and marks the item complete.

Because Someday is an inbox, **prefer `note` when Mike is at a computer** — a
someday item captured straight into its thread never needs draining. Filing to
`Work - Someday` from a session is not wrong, just a round trip. Mention the
option once; don't argue about it.

Reminders.app is otherwise *active memory* — `Work - Now` and `Work - Soon` hold
only things that need doing, so nothing dateless should accumulate there.

If the triage is genuinely ambiguous (e.g. "remind me to email Alex" where
Alex could be a colleague or a friend), ask rather than guess — a reminder
filed to the wrong list is worse than one clarifying question.

Ignore lists that are test artifacts or unrelated to triage (e.g.
`Assistant-Test`) — never file into them.

### 3. Link it to a vault thread

Grep the vault for a thread this obligation belongs to:

```bash
ls ~/workspace/knowledge-vault/main/threads/
grep -ril "<keyword from the request>" ~/workspace/knowledge-vault/main/threads/
```

Reads come from the `main` worktree, and agent writes never touch it — every agent
write lands via a squash-merged PR from a private worktree — so this always works.
Mike also writes there by hand; his uncommitted files are his to commit. There is no busy case and nothing to retry. Run
`~/workspace/knowledge-vault/main/bin/vault-write read` first if you want the read
worktree fast-forwarded.

**Still never create an unlinked reminder to get unstuck.** `recall` finds
obligations by grepping bodies for `thread: <slug>`, so a missing marker makes the
reminder invisible to the thread it belongs to — breaking the property the system
is built on, in a way he will not notice until he needs it. If no thread matches,
say so and ask; do not quietly file it bare.

If one matches, add `thread: <slug>` as the first line of the reminder `body`
(above any context links). `recall` finds reminders by grepping bodies for that
marker, so the exact `thread: ` prefix matters.

If nothing matches, **create the reminder anyway** and mention that no thread
covers it. That absence is information — it often means the thinking behind the
obligation was never deposited, and he may want `note`. It is never an error and
must never block the write.

Only link, never create: this skill does not write thread notes. It reads the
vault and nothing more; a genuinely needed vault write goes through `note` (or
`bin/vault-write begin` / `publish`), never by editing the `main` worktree.

### 4. Create the reminder

Build the due date arithmetically — `date "July 28, 2026 10:00 AM"` string
literals are locale-dependent and break on machines with non-US formats:

```bash
osascript <<'EOF'
set d to current date
set year of d to 2026
set month of d to 7
set day of d to 28
set time of d to 10 * hours -- 10:00; add `+ 30 * minutes` for half hours
tell application "Reminders"
    tell list "Work - Now"
        make new reminder with properties {name:"Follow up on PR reviews", body:"thread: search-relevance" & linefeed & "Context links here", remind me date:d}
    end tell
end tell
EOF
```

For a reminder with no trigger time, omit `remind me date` (and the date
arithmetic) entirely. Resolve relative dates ("tomorrow", "Friday") against
today's date before writing the script.

### 5. Verify

Read the reminder back and confirm it landed in the intended list. Query by
name — `last reminder of list` returns an arbitrary item, not the newest:

```bash
osascript -e 'tell application "Reminders" to get {name, remind me date} of (reminders of list "Work - Now" whose name is "Follow up on PR reviews")'
```

Then tell the user: the list it went to, the title, the trigger time, and which
thread it was linked to (or that none matched).

## Checks

- The target list came from the live list query, not from memory.
- The read-back shows the expected title and (if set) the expected date.
- If a thread matched, the body's first line is `thread: <slug>`.
- The user was told which list was chosen and why, so a bad triage is
  immediately visible.

## Stop Conditions

- Reminder text missing → ask, don't invent a title.
- Work vs. personal genuinely ambiguous → ask (one question, offer your
  best guess as the default).
- Asked to create more than ~5 reminders at once → confirm the batch before
  writing.

## Recovery

- **`osascript` permission error** (`Not authorized to send Apple events`):
  the terminal lacks Automation access. Tell the user to enable it in
  System Settings → Privacy & Security → Automation → (their terminal) →
  Reminders, then retry once.
- **Target list not found**: re-read the live list names; pick the closest
  match if it's an obvious rename, otherwise fall back to the default
  `Reminders` list and tell the user where it went.
- **Vault missing or unreadable**: skip the thread link entirely and create the
  reminder. The reminder is the deliverable; the link is an enhancement.
- **Date arithmetic errors**: drop the due date, create the reminder without
  one, and tell the user the trigger time couldn't be set.
- If creation fails twice for the same reason, stop and report the exact
  osascript error rather than retrying variations.
