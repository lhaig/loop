# Claude Code Skills

Custom skills for [Claude Code](https://claude.ai/code) that help with planning and executing software projects.

## Skills Included

### PRD Generator (`/prd`)

Creates detailed Product Requirements Documents through interactive Q&A.

**Usage:**
```
/prd
```

**What it does:**
1. Asks 3-5 clarifying questions with lettered options (answer like "1A, 2C, 3B")
2. Generates a structured PRD with user stories, functional requirements, and acceptance criteria
3. Saves to `.planning/prd-[feature-name].md`

### norman (`/norman`)

Executes large projects (10+ tasks) with crash recovery and session persistence.

**Usage:**
```
norman plan          # Interactive planning session
norman import [path] # Generate tasks from a PRD
norman               # Continue executing tasks
norman status        # Check progress
norman verify        # Validate against requirements
norman reset         # Start fresh
```

**What it does:**
- Breaks projects into small, atomic tasks with dependencies
- Executes each task in an isolated subagent (fresh context per task)
- Commits after each task (safe checkpoints)
- Tracks state in `.norman/` files so you can resume after crashes or session ends

## Recommended Workflow

**For new features:**
```
/prd                    # Plan the feature, create requirements
norman import             # Convert PRD to executable tasks
norman                    # Execute until done
norman verify             # Validate against PRD
```

**When you already know the tasks:**
```
norman plan               # Discuss and plan interactively
norman                    # Execute until done
```

## Installation

1. Clone this repo (or copy the skill folders) into your Claude Code skills directory:
   ```bash
   # Find your skills directory
   # Usually: ~/.claude/skills/ or configured in Claude Code settings

   cp -r norman/ ~/.claude/skills/
   cp -r prd/ ~/.claude/skills/
   ```

2. Restart Claude Code or reload skills

## File Structure

```
skills/
  norman/
    SKILL.md        # norman skill definition
    subagents.md    # Agent type reference (read by Haiku classifier)
  prd/
    SKILL.md        # PRD skill definition
```

**PRD creates:**
```
.planning/
  prd-[feature].md  # Requirements document
```

**Norman creates:**
```
.norman/
  tasks.md          # Task list with status and dependencies
  progress.md       # Execution log and discovered patterns
  config.md         # Project config (commands, limits)
```

### Statusline

A status bar that shows context usage, model, cost, and git branch at the bottom of your terminal.

```
[Opus] [####------] 42% | $0.35 | main
```

**Installation:**
```bash
cp statusline/statusline.sh ~/.claude/statusline.sh
chmod +x ~/.claude/statusline.sh
```

Add to `~/.claude/settings.json`:
```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.sh"
  }
}
```

See [statusline/README.md](statusline/README.md) for details.

## Requirements

- [Claude Code](https://claude.ai/code) CLI

## License

MIT License - see [LICENSE](LICENSE) for details.
