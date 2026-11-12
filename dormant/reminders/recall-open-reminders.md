# recall: open reminders (dormant)

Removed from `skills/recall/SKILL.md` on 2026-11-11. It was step 5, between
"4. Hop 3" and "Git history" (which became step 5). Restore it there and
renumber.

## Step text

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


## Other pieces removed from recall

- Description: "…his own daily logs and raw dumps, and any open reminders tied
  to them." and "…or to create reminders (use `remind`)."
- Output-shape header example included `2 open reminders ·`.
- Closing-observation example: `- The open reminder "chase PROJ-5059" names a
  ticket this thread's Next move no longer mentions.`
- Recovery bullet:

- **`osascript` permission error** (`Not authorized to send Apple events`):
  report that reminders could not be read, deliver the thread material anyway.
  Partial recall beats no recall.
