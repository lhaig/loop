---
name: loop
description: "Autonomous project execution with crash recovery. Triggers on: loop, start loop, continue loop, run the loop, set up project, resume project, loop plan, loop import, loop verify. For large features (10+ tasks) with file-based state that persists across sessions."
---

# Loop - Large Project Execution

Execute large projects with crash recovery and session persistence.

---

## Overview

Loop uses **local files** to track state and **subagents for each task**, so you can:
- Resume after crashes or session ends
- Have git commits as safe checkpoints
- Keep context fresh (each task runs in isolated subagent)

**Architecture:**
- **Main agent** = Orchestrator (reads state, spawns subagents, updates files)
- **Subagents** = Workers (execute one task each with fresh context)

**Three-tier model strategy:**
- **Haiku** = Orchestrator glue (classify tasks, gather context, parse results, compress progress)
- **Sonnet** = Default worker (implement most tasks)
- **Opus** = Heavy lifter (complex architecture, security, debugging, escalation from failed Sonnet)

**Four modes:**
1. **Plan** - Interactive planning session that generates the task file
2. **Import** - Generate tasks from an existing PRD or requirements document
3. **Continue** - Execute tasks via subagents until done or stopped
4. **Verify** - Validate implementation against original PRD/requirements

### Recommended Workflow

**For new features in existing projects:**
```
loop plan              # Discuss and plan
continue loop          # Execute tasks
```

**End-to-end with PRD skill:**
```
/prd                   # Create requirements document
loop from-prd          # Import the PRD as tasks
continue loop          # Execute tasks
loop verify            # Validate against PRD
```

---

## File Structure

**CRITICAL:** All state MUST live in a `.loop/` folder in the project root. NEVER store loop state anywhere else.

```
.loop/
  tasks.md      # Task list with status and dependencies
  progress.md   # Append-only execution log
  config.md     # Project config and thresholds
```

### Config File (.loop/config.md)

```markdown
# Loop Config

## Session Limits
max_tasks_per_session: 15
warn_at_tasks: 12

## Project
name: [Project Name]
repo: [repo path or URL]
created: [date]

## Commands (customize per project)
typecheck: npm run typecheck
lint: npm run lint
test: npm test

## Source
prd_path: [path to PRD if imported, or empty]

## Subagent Defaults
default_subagent: general-purpose

## Model Strategy
default_model: sonnet
complex_model: opus
quick_model: haiku
auto_escalate: true
progress_compress_after: 10
```

---

## Mode 1: Plan (Interactive Planning)

Start with: "loop plan" or "plan a loop"

### Phase 1: Discovery

Ask the user questions to understand scope using AskUserQuestion with structured options:

- **What are you building?** (new feature / refactor / bug fix / migration)
- **What's the scope?** (single file / multiple files / cross-cutting / full module)
- **What areas of the codebase?** (explore to identify, ask to confirm)
- **Existing patterns to follow?** (search for similar implementations)
- **Constraints?** (backward compat, performance, external deps)

Keep asking until you have enough detail to break into concrete tasks.

### Phase 2: Task Breakdown

Break into logical units, sequence by dependencies, size each task for one focused session. Present for review:

```
Phase 1: Foundation (3 tasks)
1. [Task] - [why it's first]
2. [Task] (needs: 1)
3. [Task] (needs: 1)

Phase 2: Core (4 tasks)
4. [Task] (needs: 2, 3)
...

Does this look right? Any tasks to add, remove, or reorder?
```

### Phase 3: Generate Files

Once approved, create `.loop/` directory with `tasks.md`, `config.md` (auto-detect project commands from package.json/Makefile/pyproject.toml), and `progress.md`. Commit with `chore: initialize loop for [project name]`.

---

## Mode 2: Import (From PRD/Requirements)

Start with: "loop import [path]" or "loop from-prd"

### Supported Formats

- **PRD files** - `.planning/prd-*.md` (from the prd skill)
- **Requirements docs** - Any markdown with user stories or requirements
- **Task lists** - Existing markdown checklists

### Process

1. **Read the document** — If no path provided, search `.planning/prd-*.md` and ask user which to import
2. **Extract tasks** — Parse user stories (US-001), functional requirements (FR-1), and acceptance criteria. Transform each into a task.
3. **Analyze dependencies** — Order by explicit deps, logical sequence (schema > API > UI), and cross-references
4. **Present for review** — Show extracted tasks grouped by phase, list excluded non-goals
5. **Generate files** — Same as Plan mode Phase 3. Store PRD source path in `.loop/config.md` under `## Source`

---

## Mode 3: Continue

Start with: "continue loop", "run the loop", or just "loop"

### Step 0: Initialize Session

Read `.loop/config.md`, initialize `session_tasks_completed = 0`, extract session limits.

### Step 1: Read State

Read `.loop/tasks.md`, `.loop/progress.md`, and `.loop/config.md`. Parse task statuses: `[x]` done, `[ ]` pending, `[!]` blocked.

**Progress compression:** If completed tasks exceed `progress_compress_after`, spawn a Haiku agent to compress progress.md into ~100 lines of deduplicated patterns, key decisions, and last 3 full entries.

### Step 2: Find Next Task

Select first pending task(s) where all dependencies are complete. If none ready: report completion or what's blocking.

### Step 3: Classify and Prepare (Haiku)

Spawn a Haiku agent to classify each ready task. It should:
1. Search the codebase for relevant files (Glob/Grep)
2. Read the most relevant files (max 5)
3. Return: `MODEL: [sonnet|opus]`, `SUBAGENT: [agent type]`, `FILES: [paths]`, `CONTEXT: [summary]`

Use the classification guide in `subagents.md`. If the task has an explicit `(model: opus)` tag, skip classification but still gather context. Classify multiple ready tasks in parallel.

### Step 3.5: Gather Project Rules (Once Per Session)

**CRITICAL:** Subagents do NOT inherit CLAUDE.md. Before spawning any worker, read and cache:
1. Project `CLAUDE.md` (tech stack rules, forbidden libraries, conventions)
2. Global `~/.claude/CLAUDE.md` (user preferences)
3. Referenced context files (e.g., `context/tech-stack.md`)

Extract key rules into a `project_rules` block included in every subagent prompt.

### Step 4: Spawn Worker Subagent

Use Task tool with classified model and subagent type. The prompt MUST include:
- **Project Rules** from Step 3.5 (non-negotiable)
- **Task description** from tasks.md
- **Relevant files** and **current state** from Step 3
- **Patterns & learnings** from progress.md
- **Project commands** for typecheck/lint/test
- Instructions to report: DONE/FAILED/BLOCKED, files changed, summary, and any `PATTERN: [category] - [description]` discoveries
- Rules: do NOT commit, do NOT modify `.loop/` files

Spawn multiple independent workers in parallel if multiple tasks are ready.

### Step 5: Process Result

**DONE:** Update tasks.md `[ ]` → `[x]`, append to progress.md (date, task, model, changes, patterns), commit with `feat([scope]): [description]`. If subagent reported broadly useful patterns, offer to promote to CLAUDE.md.

**FAILED:** If `auto_escalate` is enabled and task ran on sonnet, auto-retry on opus with failure context. If opus also fails or escalation disabled, mark `[!]` in tasks.md, log failure, ask user: retry/skip/stop.

**BLOCKED:** Present subagent's question to user, get answer, re-spawn with additional context.

### Step 6: Continue Loop

Increment `session_tasks_completed`. Check limits:
- At `warn_at_tasks`: show warning, continue
- At `max_tasks_per_session`: stop, report progress, recommend fresh session

Then: more tasks ready → Step 2. Nothing ready but some pending → report blockers. All done → report completion, suggest `loop verify`.

Auto-continue until: task fails, all complete, user interrupts, subagent blocked, or session limit reached.

---

## Mode 4: Verify

Start with: "loop verify"

### Process

1. **Locate requirements** — Check `prd_path` in `.loop/config.md`, search `.planning/prd-*.md`, or ask user. If no PRD exists, fall back to task-based verification using tasks.md descriptions.

2. **Extract requirements (Haiku)** — Parse the requirements doc into a structured checklist: user stories with acceptance criteria, functional requirements, non-functional requirements, explicit constraints. Skip non-goals.

3. **Verify each requirement (Opus)** — Spawn a code-reviewer agent on opus. For each requirement: search codebase for implementation, read code, check acceptance criteria, run tests. Report each as PASS, FAIL, or PARTIAL with evidence.

4. **Present results** — Show pass/partial/fail counts and details.

5. **Handle gaps** — Offer via AskUserQuestion:
   - **Create tasks for gaps** — Add fix tasks to `.loop/tasks.md` as a new phase, continue loop
   - **Accept as-is** — Log results, write `.loop/VERIFICATION.md`, mark loop as verified
   - **Re-verify specific items** — Re-check individual requirements after manual fixes

Save verification report to `.loop/VERIFICATION.md` and commit.

---

## Mode 5: Reset

Start with: "loop reset"

Check current state (incomplete tasks, uncommitted changes), then offer:
- **Archive** — Move `.loop/` to `.loop-archive/[project-name]-[date]/`, ready for new project
- **Delete** — Remove `.loop/` entirely, commit removal
- **Cancel**

---

## Commands

| Command | Action |
|---------|--------|
| `loop` / `continue loop` | Execute next task(s) |
| `loop plan` | Interactive planning session |
| `loop import [path]` | Generate tasks from PRD or requirements doc |
| `loop from-prd` | Search for and import from recent PRD |
| `loop status` | Show progress summary |
| `loop task [N]` | Execute specific task |
| `loop skip [N]` | Skip a blocked task |
| `loop add [desc]` | Add new task |
| `loop pause` | Stop after current task |
| `loop verify` | Verify implementation against PRD/requirements |
| `loop reset` | Clear current project and start fresh |
| `loop learnings` | Review and promote patterns to CLAUDE.md |

---

## Related Skills

```
/prd              →  loop import  →  continue loop  →  loop verify
 (requirements)      (plan tasks)    (execute)         (validate)
```

- **PRD** (`/prd`) — Create requirements documents, saved to `.planning/`.
- **Loop** works standalone too — use `loop plan` when you don't need a PRD.

---

## Status Report

When asked for status, show: project name, progress (done/total/percent), models used, verification status, source PRD, then list completed/ready/blocked/remaining tasks. When all complete, suggest `loop verify`, `loop learnings`, or `loop reset`.

---

## Recovery

- **Session crashed** — Just say "continue loop", state is in files
- **Task partially complete** — Check git status, either commit partial progress or `git checkout .` and retry
- **Wrong task executed** — Revert commit, mark task pending, continue
- **Subagent failed/timed out** — Check changes, commit or reset, mark `[!]`, continue or retry
- **Context getting long** — After ~15-20 tasks, recommend fresh session. All state persists in `.loop/` files.

---

## Task File Syntax

```markdown
# Project: [Name]
> [One-line description]
Started: [date]
Status: in-progress

---

## Tasks

### Phase 1: Foundation
- [ ] 1. First task description
- [ ] 2. Second task (needs: 1)
- [ ] 3. Third task (needs: 1)

### Phase 2: Core Features
- [ ] 4. Fourth task (needs: 2, 3)

---

## Notes
[Important context, decisions, constraints]
```

**Syntax:** `- [ ]` pending, `- [x]` completed, `- [!]` failed/blocked, `(needs: 1, 2)` dependencies, `(model: opus)` force model.

---

## Subagent Architecture

Every task runs in a subagent for fresh context and isolation. The orchestrator reads/updates `.loop/` files, spawns subagents, processes results, and commits. Workers implement tasks and report results but do NOT commit or modify `.loop/` files.

**Specialized agents:** The full list of 25+ agent types with classification guidance is in `subagents.md` (same directory as this file).

**Parallel execution:** If multiple tasks are ready with no dependency conflicts, classify and execute in parallel.

**Auto-escalation:** Sonnet failure → auto-retry on Opus (if enabled). Opus failure → mark `[!]`, ask user.

---

## Knowledge Persistence

- **Session learnings** (`.loop/progress.md`) — Patterns discovered during execution, passed to each subagent via the "Patterns & Learnings" section
- **Permanent learnings** (`CLAUDE.md`) — Broadly useful patterns promoted from progress.md. Subagents report patterns as `PATTERN: [category] - [description]`. Orchestrator always adds to progress.md and offers CLAUDE.md promotion if broadly useful.
- **Manual review** (`loop learnings`) — Review all patterns, grouped by category, choose which to promote
