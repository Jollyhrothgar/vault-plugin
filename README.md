# vault

A Claude Code plugin that keeps thread notes in a personal knowledge vault, so
context survives a task switch.

| Skill | What it does |
| --- | --- |
| `/vault:note` | Capture what you are thinking on the way out of a task |
| `/vault:recall` | Get it back on the way in |
| `/vault:remind` | Link a macOS reminder to a thread |
| `/vault:prune` | Sweep stale threads |

## Requirements

**As published, this plugin works only with Mike's own vault.** The skills
read and write a git repository checked out at
`~/workspace/knowledge-vault/main` and call that repository's
`bin/vault-write` helper to publish changes. Both the path and the helper are
assumed, not shipped here. The vault is a private repository, so for anyone
else the skills have nothing to work on.

What the skills expect inside the vault:

- `threads/`, one note per line of work
- `topics.md`, an index of threads
- `updates/`, dated update notes
- Anything else is Mike's own space, which he organizes and commits in `main`
  by hand: daily logs, raw dumps, voice memo transcripts. The skills find these
  by searching the vault, read them, and never write them. Frontmatter
  `author: claude` marks the agent-written notes among them.
- `bin/vault-write`, which writes in a temporary worktree and publishes
  through a pull request

`remind` also needs macOS, for Reminders.

## Install

```bash
claude plugin marketplace add Jollyhrothgar/vault-plugin
claude plugin install vault@vault-plugin
```

To try a working copy without installing: `claude --plugin-dir .`
