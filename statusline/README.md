# Statusline

A status bar for Claude Code that shows context usage, model name, cost, and git branch at the bottom of your terminal.

```
[Opus] [####------] 42% | $0.35 | main
```

## Installation

1. Copy the script to your Claude config directory:
   ```bash
   cp statusline/statusline.sh ~/.claude/statusline.sh
   chmod +x ~/.claude/statusline.sh
   ```

2. Add to `~/.claude/settings.json`:
   ```json
   {
     "statusLine": {
       "type": "command",
       "command": "~/.claude/statusline.sh"
     }
   }
   ```

3. Restart Claude Code.

## What it shows

| Field | Source |
|-------|--------|
| Model name | `model.display_name` (Opus, Sonnet, Haiku) |
| Context usage | `context_window.used_percentage` with a progress bar |
| Session cost | `cost.total_cost_usd` |
| Git branch | Current branch from the project directory |

## Requirements

- `jq` (for parsing JSON input)
- `git` (for branch display)
