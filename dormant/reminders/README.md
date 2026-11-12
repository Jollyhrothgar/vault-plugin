# Reminders integration (dormant)

Switched off on 2026-11-11. The vault is now a personal knowledge vault, and the
work-oriented Reminders.app lists (`Work - Now`, `Work - Soon`, `Work - Someday`)
and their routing rules no longer fit. Nothing in `dormant/` is loaded by Claude
Code: only `skills/` is discovered.

What's here:

| File | Was | Did |
| --- | --- | --- |
| `remind/SKILL.md` | `skills/remind/SKILL.md` | `/vault:remind`: triaged a reminder into a Reminders list and linked it to a thread with `thread: <slug>` in the body |
| `note-someday-drain.md` | step 9 of `skills/note/SKILL.md` | Drained the `Work - Someday` list into threads' `## Someday` sections on every `note` run |
| `recall-open-reminders.md` | step 5 of `skills/recall/SKILL.md` | Listed open reminders whose body named the thread |

## Bringing it back

1. Rethink the list routing in `remind/SKILL.md` for personal use before restoring
   it. That routing is the main reason the integration was switched off.
2. `git mv dormant/reminders/remind skills/remind`.
3. Paste each step back where its file says it came from, and renumber the steps after it.
4. Restore the cross-references: the "use `remind`" lines in the `note` and `recall`
   descriptions and in note's stop conditions, recall's header and observation
   examples, the plugin description in both `.claude-plugin/*.json` files, and the
   README table row. `git log -p` on the commit that added this file shows every
   removed line.
