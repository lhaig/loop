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

### Loop (`/loop`)

Executes large projects (10+ tasks) with crash recovery and session persistence.

**Usage:**
```
loop plan          # Interactive planning session
loop import [path] # Generate tasks from a PRD
loop               # Continue executing tasks
loop status        # Check progress
loop verify        # Validate against requirements
loop reset         # Start fresh
```

**What it does:**
- Breaks projects into small, atomic tasks with dependencies
- Executes each task in an isolated subagent (fresh context per task)
- Commits after each task (safe checkpoints)
- Tracks state in `.loop/` files so you can resume after crashes or session ends

## Recommended Workflow

**For new features:**
```
/prd                    # Plan the feature, create requirements
loop import             # Convert PRD to executable tasks
loop                    # Execute until done
loop verify             # Validate against PRD
```

**When you already know the tasks:**
```
loop plan               # Discuss and plan interactively
loop                    # Execute until done
```

## Installation

1. Clone this repo (or copy the skill folders) into your Claude Code skills directory:
   ```bash
   # Find your skills directory
   # Usually: ~/.claude/skills/ or configured in Claude Code settings

   cp -r loop/ ~/.claude/skills/
   cp -r prd/ ~/.claude/skills/
   ```

2. Restart Claude Code or reload skills

## File Structure

```
skills/
  loop/
    SKILL.md        # Loop skill definition
    subagents.md    # Agent type reference (read by Haiku classifier)
  prd/
    SKILL.md        # PRD skill definition
```

**PRD creates:**
```
.planning/
  prd-[feature].md  # Requirements document
```

**Loop creates:**
```
.loop/
  tasks.md          # Task list with status and dependencies
  progress.md       # Execution log and discovered patterns
  config.md         # Project config (commands, limits)
```

## Requirements

- [Claude Code](https://claude.ai/code) CLI

## License

MIT License - see [LICENSE](LICENSE) for details.
