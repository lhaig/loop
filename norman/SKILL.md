---
name: norman
description: "Autonomous project execution with crash recovery. Triggers on: norman, start norman, continue norman, run the norman, set up project, resume project, norman plan, norman import, norman verify. For large features (10+ tasks) with file-based state that persists across sessions."
---

# Norman - Large Project Execution

Execute large projects with crash recovery and session persistence.

---

## Overview

Norman uses **local files** to track state and **subagents for each task**, so you can:
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
norman plan              # Discuss and plan
continue norman          # Execute tasks
```

**End-to-end with PRD skill:**
```
/prd                   # Create requirements document
norman from-prd          # Import the PRD as tasks
continue norman          # Execute tasks
norman verify            # Validate against PRD
```

---

## File Structure

**CRITICAL:** All state MUST live in a `.norman/` folder in the project root. NEVER store norman state anywhere else.

```
.norman/
  tasks.md      # Task list with status and dependencies
  progress.md   # Append-only execution log
  config.md     # Project config and thresholds
```

### Config File (.norman/config.md)

```markdown
# Norman Config

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

Start with: "norman plan" or "plan a norman"

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

Once approved, create `.norman/` directory with `tasks.md`, `config.md` (auto-detect project commands from package.json/Makefile/pyproject.toml), and `progress.md`. Commit with `chore: initialize norman for [project name]`.

---

## Mode 2: Import (From PRD/Requirements)

Start with: "norman import [path]" or "norman from-prd"

### Supported Formats

- **PRD files** - `.planning/prd-*.md` (from the prd skill)
- **Requirements docs** - Any markdown with user stories or requirements
- **Task lists** - Existing markdown checklists

### Process

1. **Read the document** — If no path provided, search `.planning/prd-*.md` and ask user which to import
2. **Extract tasks** — Parse user stories (US-001), functional requirements (FR-1), and acceptance criteria. Transform each into a task.
3. **Analyze dependencies** — Order by explicit deps, logical sequence (schema > API > UI), and cross-references
4. **Present for review** — Show extracted tasks grouped by phase, list excluded non-goals
5. **Generate files** — Same as Plan mode Phase 3. Store PRD source path in `.norman/config.md` under `## Source`

---

## Mode 3: Continue

Start with: "continue norman", "run the norman", or just "norman"

### Step 0: Initialize Session

Read `.norman/config.md`, initialize `session_tasks_completed = 0`, extract session limits.

### Step 1: Read State

Read `.norman/tasks.md`, `.norman/progress.md`, and `.norman/config.md`. Parse task statuses: `[x]` done, `[ ]` pending, `[!]` blocked.

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
- Rules: do NOT commit, do NOT modify `.norman/` files

Spawn multiple independent workers in parallel if multiple tasks are ready.

### Step 5: Process Result

**DONE:** Update tasks.md `[ ]` → `[x]`, append to progress.md (date, task, model, changes, patterns), commit with `feat([scope]): [description]`. If subagent reported broadly useful patterns, offer to promote to CLAUDE.md.

**FAILED:** If `auto_escalate` is enabled and task ran on sonnet, auto-retry on opus with failure context. If opus also fails or escalation disabled, mark `[!]` in tasks.md, log failure, ask user: retry/skip/stop.

**BLOCKED:** Present subagent's question to user, get answer, re-spawn with additional context.

### Step 6: Continue Norman

Increment `session_tasks_completed`. Check limits:
- At `warn_at_tasks`: show warning, continue
- At `max_tasks_per_session`: stop, report progress, recommend fresh session

Then: more tasks ready → Step 2. Nothing ready but some pending → report blockers. All done → report completion, suggest `norman verify`.

Auto-continue until: task fails, all complete, user interrupts, subagent blocked, or session limit reached.

---

## Mode 4: Verify

Start with: "norman verify"

### Process

1. **Locate requirements** — Check `prd_path` in `.norman/config.md`, search `.planning/prd-*.md`, or ask user. If no PRD exists, fall back to task-based verification using tasks.md descriptions.

2. **Extract requirements (Haiku)** — Parse the requirements doc into a structured checklist: user stories with acceptance criteria, functional requirements, non-functional requirements, explicit constraints. Skip non-goals.

3. **Verify each requirement (Opus)** — Spawn a code-reviewer agent on opus. For each requirement: search codebase for implementation, read code, check acceptance criteria, run tests. Report each as PASS, FAIL, or PARTIAL with evidence.

4. **Present results** — Show pass/partial/fail counts and details.

5. **Handle gaps** — Offer via AskUserQuestion:
   - **Create tasks for gaps** — Add fix tasks to `.norman/tasks.md` as a new phase, continue norman
   - **Accept as-is** — Log results, write `.norman/VERIFICATION.md`, mark norman as verified
   - **Re-verify specific items** — Re-check individual requirements after manual fixes

Save verification report to `.norman/VERIFICATION.md` and commit.

---

## Mode 5: Reset

Start with: "norman reset"

Check current state (incomplete tasks, uncommitted changes), then offer:
- **Archive** — Move `.norman/` to `.norman-archive/[project-name]-[date]/`, ready for new project
- **Delete** — Remove `.norman/` entirely, commit removal
- **Cancel**

---

## Commands

| Command | Action |
|---------|--------|
| `norman` / `continue norman` | Execute next task(s) |
| `norman plan` | Interactive planning session |
| `norman import [path]` | Generate tasks from PRD or requirements doc |
| `norman from-prd` | Search for and import from recent PRD |
| `norman status` | Show progress summary |
| `norman task [N]` | Execute specific task |
| `norman skip [N]` | Skip a blocked task |
| `norman add [desc]` | Add new task |
| `norman pause` | Stop after current task |
| `norman verify` | Verify implementation against PRD/requirements |
| `norman reset` | Clear current project and start fresh |
| `norman learnings` | Review and promote patterns to CLAUDE.md |

---

## Related Skills

```
/prd              →  norman import  →  continue norman  →  norman verify
 (requirements)      (plan tasks)    (execute)         (validate)
```

- **PRD** (`/prd`) — Create requirements documents, saved to `.planning/`.
- **Norman** works standalone too — use `norman plan` when you don't need a PRD.

---

## Status Report

When asked for status, show: project name, progress (done/total/percent), models used, verification status, source PRD, then list completed/ready/blocked/remaining tasks. When all complete, suggest `norman verify`, `norman learnings`, or `norman reset`.

---

## Recovery

- **Session crashed** — Just say "continue norman", state is in files
- **Task partially complete** — Check git status, either commit partial progress or `git checkout .` and retry
- **Wrong task executed** — Revert commit, mark task pending, continue
- **Subagent failed/timed out** — Check changes, commit or reset, mark `[!]`, continue or retry
- **Context getting long** — After ~15-20 tasks, recommend fresh session. All state persists in `.norman/` files.

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

Every task runs in a subagent for fresh context and isolation. The orchestrator reads/updates `.norman/` files, spawns subagents, processes results, and commits. Workers implement tasks and report results but do NOT commit or modify `.norman/` files.

**Specialized agents:** The full list of 25+ agent types with classification guidance is in `subagents.md` (same directory as this file).

**Parallel execution:** If multiple tasks are ready with no dependency conflicts, classify and execute in parallel.

**Auto-escalation:** Sonnet failure → auto-retry on Opus (if enabled). Opus failure → mark `[!]`, ask user.

---

## Knowledge Persistence

- **Session learnings** (`.norman/progress.md`) — Patterns discovered during execution, passed to each subagent via the "Patterns & Learnings" section
- **Permanent learnings** (`CLAUDE.md`) — Broadly useful patterns promoted from progress.md. Subagents report patterns as `PATTERN: [category] - [description]`. Orchestrator always adds to progress.md and offers CLAUDE.md promotion if broadly useful.
- **Manual review** (`norman learnings`) — Review all patterns, grouped by category, choose which to promote
