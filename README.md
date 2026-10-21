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

- A vault checked out at `~/workspace/knowledge-vault/main`, with its
  `bin/vault-write` helper.
- macOS, for `remind`.

## Install

```bash
claude plugin marketplace add Jollyhrothgar/personal_ai_config
claude plugin install vault@jollyhrothgar
```

To try a working copy without installing: `claude --plugin-dir .`
