---
name: loop
description: "Autonomous project execution with crash recovery. Triggers on: loop, start loop, continue loop, run the loop, set up project, resume project, loop plan, loop import, loop verify, loop statusline. For large features (10+ tasks) with file-based state that persists across sessions."
---

# Loop - Large Project Execution

Execute large projects (30+ tasks) with crash recovery and session persistence.

---

## Overview

Loop uses **local files** to track state and **subagents for each task**, so you can:
- Resume after crashes or session ends
- See exactly what's done and what's next
- Have git commits as safe checkpoints
- Keep context fresh (each task runs in isolated subagent)

**Architecture:**
- **Main agent** = Orchestrator (reads state, spawns subagents, updates files)
- **Subagents** = Workers (execute one task each with fresh context)

**Three-tier model strategy:**
- **Haiku** = Orchestrator glue (classify tasks, gather context, parse results, compress progress)
- **Sonnet** = Default worker (implement most tasks - fast and capable)
- **Opus** = Heavy lifter (complex architecture, security, debugging, escalation from failed Sonnet)

**Five modes:**
1. **Plan** - Interactive planning session that generates the task file
2. **Import** - Generate tasks from an existing PRD or requirements document
3. **Quick Setup** - Manual setup when you already know the tasks
4. **Continue** - Execute tasks via subagents until done or stopped
5. **Verify** - Validate implementation against original PRD/requirements

### Recommended Workflow

**For new features in existing projects:**
```
loop plan              # Discuss and plan
continue loop          # Execute tasks
```

**When you already have a PRD:**
```
loop import tasks/prd-my-feature.md   # Generate from PRD
continue loop                          # Execute tasks
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

All state lives in a `.loop/` folder in the project root:

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
max_tasks_per_session: 15    # Pause and recommend restart after N tasks
warn_at_tasks: 12            # Show warning at this count

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
# Override per phase if needed:
# phase_1_subagent: sql-pro
# phase_3_subagent: test-automator

## Model Strategy
default_model: sonnet           # most tasks run on sonnet (fast, capable)
complex_model: opus             # complex/security/architecture tasks
quick_model: haiku              # orchestrator glue (classify, parse, compress)
auto_escalate: true             # retry sonnet failures on opus automatically
progress_compress_after: 10     # use haiku to compress progress.md after N tasks
```

The orchestrator reads this config at the start of each session and tracks tasks completed in the current session.

---

## Mode 1: Plan (Interactive Planning)

Start with: "loop plan" or "plan a loop"

This mode guides you through a structured planning conversation and automatically generates all `.loop/` files at the end.

### Phase 1: Discovery

Ask the user a series of questions to understand the scope. Use the AskUserQuestion tool with structured options where possible.

**Essential questions:**

1. **What are you building?**
   - New feature from scratch
   - Refactoring existing code
   - Bug fix / technical debt
   - Migration / upgrade

2. **What's the scope?**
   - Single file/component
   - Multiple related files
   - Cross-cutting (touches many areas)
   - Full system/module

3. **What areas of the codebase are involved?**
   - [Explore to identify relevant directories/files]
   - Ask user to confirm or add

4. **Are there existing patterns to follow?**
   - [Search for similar implementations in codebase]
   - Note any patterns found

5. **What are the constraints?**
   - Backward compatibility needed?
   - Performance requirements?
   - External dependencies?
   - Deadline or priority?

**Keep asking until you have enough detail to break into concrete tasks.**

### Phase 2: Task Breakdown

Based on discovery, create a task breakdown:

1. **Analyze the work** - Break into logical units
2. **Sequence by dependencies** - What must come first?
3. **Size appropriately** - Each task completable in one focused session
4. **Present for review**:

```
Here's my proposed breakdown:

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

Once the user approves, automatically:

1. Create `.loop/` directory
2. Generate `tasks.md` from approved breakdown
3. Generate `config.md` by detecting project commands
4. Initialize `progress.md`
5. Commit the setup

Then show:
```
Loop initialized from planning session!

Project: [name]
Tasks: [count] across [phases] phases

To start: say "continue loop"
To check status: say "loop status"
```

---

## Mode 2: Import (From PRD/Requirements)

Start with: "loop import [path]" or "loop from-prd"

This mode reads an existing requirements document and generates the task list.

### Supported Formats

- **PRD files** - `/tasks/prd-*.md` (from the prd skill)
- **Requirements docs** - Any markdown with user stories or requirements
- **Task lists** - Existing markdown checklists

### Step 1: Read the Document

```bash
cat [provided path or search /tasks/prd-*.md]
```

If no path provided, search for recent PRDs:
```bash
ls -lt tasks/prd-*.md 2>/dev/null | head -5
```

Ask user which to import if multiple found.

### Step 2: Extract Tasks

Parse the document for:
- **User Stories** (US-001, US-002, etc.)
- **Functional Requirements** (FR-1, FR-2, etc.)
- **Acceptance Criteria** (as task validation steps)

Transform each user story into a task:

| PRD Element | Loop Task |
|-------------|-----------|
| US-001: Title | Task description |
| Acceptance Criteria | Included in task detail |
| Dependencies mentioned | `(needs: X)` syntax |

### Step 3: Analyze Dependencies

Determine task order by:
1. Explicit dependencies in PRD
2. Logical sequence (schema → API → UI)
3. References between stories ("after US-001...")

### Step 4: Present for Review

```
I've extracted [N] tasks from [document name]:

Phase 1: Foundation
1. [From US-001] - Task description
2. [From US-002] (needs: 1)

Phase 2: Core Features
3. [From US-003] (needs: 2)
...

Non-goals from PRD (excluded):
- [Listed non-goals]

Should I create the loop files with this breakdown?
```

### Step 5: Generate Files

Same as Plan mode Phase 3 - create all `.loop/` files and commit.

**Important:** Store the PRD source path in `.loop/config.md` so verification can reference it later:

```markdown
## Source
prd_path: tasks/prd-my-feature.md
```

---

## Mode 3: Quick Setup (Manual)

Start with: "set up loop" or "loop setup"

This is the original manual mode for when you already know exactly what tasks you need.

### Step 1: Understand the Project

Ask the user:
```
What are you building or refactoring?
```

Then clarify:
- What's the end goal?
- What areas of the codebase are involved?
- Are there existing patterns to follow?
- Any constraints or requirements?

**Keep asking until you can break it into concrete tasks.**

### Step 2: Break Into Tasks

**Each task must be completable in one focused session.**

Good task size:
- Add a database migration
- Create one component
- Implement one API endpoint
- Write tests for one module
- Refactor one file/function

Too big (split these):
- "Build the dashboard" → schema, API, components, styling
- "Add authentication" → schema, middleware, UI, session handling

**Rule:** If you can't describe it in 2-3 sentences, split it.

### Step 3: Order by Dependencies

Typical order:
1. Schema/database changes
2. Core utilities/helpers
3. Backend/API logic
4. Frontend components
5. Integration/tests
6. Cleanup/polish

Tasks can run in parallel if they share the same dependencies.

### Step 4: Create Task File

Create `.loop/tasks.md`:

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
- [ ] 5. Fifth task (needs: 4)

### Phase 3: Polish
- [ ] 6. Sixth task (needs: 5)

---

## Notes

[Any important context, decisions, or constraints]
```

**Syntax:**
- `- [ ]` = pending
- `- [x]` = completed
- `- [!]` = failed/blocked (needs attention)
- `(needs: 1, 2)` = depends on tasks 1 and 2
- `(model: opus)` = force a specific model for this task (optional, overrides auto-classification)

### Step 5: Initialize Progress Log

Create `.loop/progress.md`:

```markdown
# Progress Log

## Patterns Discovered
(Add reusable patterns here as you find them)

---

## Execution Log

### [Date] - Project Setup
- Created task list with [N] tasks
- Key decisions: [list any]
---
```

### Step 6: Create Config

Create `.loop/config.md` using the format shown in the Config File section above. Auto-detect commands by checking `package.json`, `Makefile`, `pyproject.toml`, or ask user if unclear.

### Step 7: Commit and Confirm

Commit `.loop/` files with `chore: initialize loop for [project name]`. Show task count, phase count, and remind user to say "continue loop" to start.

---

## Mode 4: Continue

Start with: "continue loop", "run the loop", or just "loop"

### Step 0: Initialize Session Tracking (Orchestrator)

At the START of each session, initialize:
```
session_tasks_completed = 0
```

Read config:
```bash
cat .loop/config.md
```

Extract limits:
- `max_tasks_per_session` (default: 15)
- `warn_at_tasks` (default: 12)

### Step 1: Read State (Orchestrator)

Read `.loop/tasks.md`, `.loop/progress.md`, and `.loop/config.md`.

Parse:
- Which tasks are done `[x]`
- Which tasks are pending `[ ]`
- Which tasks are blocked `[!]`
- What dependencies are satisfied

**Progress compression (Haiku):** If completed tasks exceed `progress_compress_after` (default: 10), spawn a Haiku agent to compress progress.md into ~100 lines: deduplicated patterns, key decisions, notable issues, and last 3 task entries in full. Store as `progress_context` for subagent prompts. Below threshold, use progress.md directly.

### Step 2: Find Next Task (Orchestrator)

Select the first pending task(s) where all dependencies are complete.

If no task is ready:
- All done → Report completion
- Some blocked → Report what's blocking

If multiple tasks are ready:
- Prefer tasks in the same area as recently completed work
- Or ask user which to prioritize

### Step 3: Classify and Prepare (Haiku)

For each ready task, spawn a **Haiku agent** that classifies complexity and gathers context in a single call:

```
Task tool:
  subagent_type: "general-purpose"
  model: "haiku"
  description: "Classify task [N]"
  prompt: |
    Analyze this task and prepare context for a worker agent.

    ## Task
    [Full task description from tasks.md]

    ## Instructions

    1. Search the codebase for files likely involved in this task (use Glob and Grep)
    2. Read the most relevant files (max 5) to understand current state
    3. Classify the task complexity

    ## Classification Rules

    Return EXACTLY this format:

    MODEL: [sonnet|opus]
    SUBAGENT: [best matching agent type from the list below]
    FILES: [comma-separated list of relevant file paths]
    CONTEXT: [2-3 sentence summary of current state of the code relevant to this task]

    ## Agent Types and Classification Guide
    Read the file `subagents.md` in the loop skill directory for the full list of
    available agent types and the classification guide for choosing sonnet vs opus.
```

**Override logic:** If the task has an explicit `(model: opus)` or `(model: sonnet)` tag in tasks.md, skip classification and use that model directly. Still run Haiku for file discovery and context gathering.

If multiple independent tasks are ready, spawn classification agents **in parallel** (one Haiku call per task).

### Step 3.5: Gather Project Rules (Orchestrator)

**CRITICAL:** Subagents do NOT inherit CLAUDE.md or any project context automatically. They only see what you put in the prompt. Before spawning any worker, the orchestrator MUST read and include:

1. **Project CLAUDE.md** - Read `CLAUDE.md` from the project root (if it exists). This contains tech stack rules, forbidden libraries, coding conventions, and user preferences that ALL subagents must follow.
2. **Global CLAUDE.md** - Read `~/.claude/CLAUDE.md` (if it exists). This contains the user's global preferences across all projects.
3. **Context files** - If CLAUDE.md references context files (e.g., `context/tech-stack.md`, `context/conventions.md`), read those too.

Extract the key rules into a `project_rules` block. Focus on:
- Technology constraints (required/forbidden libraries and frameworks)
- Code style requirements
- Testing requirements
- Security requirements
- Any "NEVER do X" or "ALWAYS do Y" rules

This only needs to be done **once per session** - cache the result and include it in every subagent prompt.

### Step 4: Spawn Worker Subagent (Orchestrator)

Use the Task tool with the **model from Step 3** (or config default):

```
Task tool:
  subagent_type: [from Step 3 SUBAGENT classification]
  model: [from Step 3 MODEL classification, or task override, or config default_model]
  description: "Task [N]: [short title]"
  prompt: |
    ## Project Rules (MUST follow)
    [project_rules from Step 3.5 - tech stack, forbidden libs, conventions]
    [Include verbatim any NEVER/ALWAYS rules from CLAUDE.md]

    ## Task
    [Full task description from tasks.md]

    ## Context
    Project: [project name]
    Working directory: [path]
    Model: [model being used] (if this fails, orchestrator may retry on opus)

    ## Relevant Files
    [FILES list from Step 3 Haiku classification]

    ## Current State
    [CONTEXT summary from Step 3 Haiku classification]

    ## Patterns & Learnings
    [progress_context from Step 1 - compressed or raw]

    ## Instructions
    1. Read CLAUDE.md in the project root before starting (if you haven't already)
    2. Read the relevant files first to understand current state
    3. Implement the task following the project rules above
    4. Run type checks / linting: [project-specific command]
    5. Run tests if relevant: [project-specific command]
    6. If checks fail, fix the issues before completing

    ## When Complete
    Report back with:
    - DONE or FAILED or BLOCKED
    - Files changed (list)
    - Summary of what was implemented
    - Any issues encountered

    ## Patterns to Report
    Report any patterns you discovered that would help future tasks:
    - Project conventions (naming, structure, API shapes)
    - Existing utilities or helpers that should be reused
    - Testing patterns that worked well
    - Architecture decisions you uncovered

    Format: "PATTERN: [category] - [description]"

    ## Important
    - Do NOT commit - orchestrator handles commits
    - Do NOT modify .loop/ files - orchestrator handles state
    - If you need user input, report BLOCKED with the question
    - FOLLOW the Project Rules section above - these are non-negotiable user preferences
```

If multiple independent tasks are ready and classified, spawn workers **in parallel**.

### Step 5: Process Result (Orchestrator)

**If DONE:**
1. Update tasks.md: Change `- [ ] N.` to `- [x] N.`
2. Append to progress.md:
   ```markdown
   ### [Date] - Task [N]: [Title] [model: sonnet|opus]
   - What was done: [from subagent report]
   - Files changed: [from subagent report]
   - Patterns: [any PATTERN: lines from subagent]
   ---
   ```
3. Add any new patterns to "Patterns Discovered" section in progress.md
4. **Check for promotable patterns:**
   - If subagent reported patterns that are broadly useful (conventions, architecture, reusable approaches)
   - Ask user: "Promote this pattern to CLAUDE.md? [pattern description]"
   - If yes, append to project's `CLAUDE.md` under `## Learnings` section
5. Commit:
   ```bash
   git add -A
   git commit -m "feat([scope]): [task description]"
   ```

**If FAILED (with auto-escalation):**
1. Check if `auto_escalate` is enabled in config AND the task ran on sonnet:
   - **Yes:** Log the failure reason, then **automatically retry on opus**:
     ```
     Task [N] failed on sonnet. Auto-escalating to opus...
     ```
     Re-spawn the worker subagent with `model: "opus"`, including the failure context:
     ```
     Previous attempt on sonnet failed with: [failure reason]
     Please review what went wrong and implement correctly.
     ```
     If opus also fails → proceed to manual handling below
   - **No (already on opus, or auto_escalate disabled):**
     1. Update tasks.md: Change `- [ ] N.` to `- [!] N.`
     2. Log failure details in progress.md with `[model: X, escalated: yes/no]`
     3. Ask user: retry, skip, or stop?

**If BLOCKED:**
1. Present the subagent's question to user
2. Get answer
3. Re-spawn subagent with additional context (same model)

### Step 6: Continue Loop (Orchestrator)

After successful task completion:

```
session_tasks_completed += 1
```

**Check session limits:**

```
if session_tasks_completed >= max_tasks_per_session:
    STOP - Session limit reached
    Report: "Completed [N] tasks this session. Recommend starting fresh session."
    Report: "Progress: [done]/[total] overall. Just say 'continue loop' to resume."

elif session_tasks_completed >= warn_at_tasks:
    WARN - Approaching limit
    Show: "Completed [N] tasks this session ([M] until recommended restart)"
    Continue executing...
```

**Then check task state:**
1. If more tasks ready → Go to Step 2 (find next task)
2. If no tasks ready but some pending → Report what's blocking
3. If all done → Report completion, summarize, and suggest verification:
   ```
   All [N] tasks complete!

   Recommend running verification to check implementation against requirements.
   Say "loop verify" to validate, or "loop learnings" to review patterns first.
   ```

**Auto-continue** until:
- Task fails
- All tasks complete
- User interrupts (Ctrl+C or says "loop pause")
- Subagent reports BLOCKED
- **Session task limit reached**

---

## Mode 5: Verify

Start with: "loop verify" (also auto-suggested when all tasks complete)

Validates that the implementation actually satisfies the original PRD or requirements.

### When to Run

- **Automatically suggested** when the last task completes in Mode 4
- **Manually** at any point via `loop verify`
- **After resuming** if you want to check partial progress against requirements

### Step 1: Locate Requirements Source (Orchestrator)

Read `.loop/config.md` and look for `prd_path` under `## Source`.

**If prd_path exists:** Read that file.
**If prd_path is empty or missing:**
1. Search for PRDs: `ls -lt tasks/prd-*.md 2>/dev/null | head -5`
2. Check `.loop/tasks.md` header for any referenced source document
3. If nothing found, ask the user: "Which requirements document should I verify against?"

If no requirements doc exists at all (e.g., project was set up via `loop plan` without a PRD), fall back to **task-based verification** — verify each task's description and acceptance criteria were met.

### Step 2: Extract Requirements (Haiku)

Spawn a **Haiku agent** to parse the requirements document into a structured checklist:

```
Task tool:
  subagent_type: "general-purpose"
  model: "haiku"
  description: "Extract verification checklist"
  prompt: |
    Read the following requirements document and extract every verifiable requirement.

    ## Document
    [contents of PRD or requirements file]

    ## Instructions

    Extract ALL of the following into a structured checklist:
    1. User Stories - each one with its acceptance criteria
    2. Functional Requirements (FR-1, FR-2, etc.)
    3. Non-functional requirements (performance, security, accessibility)
    4. Explicit constraints or rules mentioned anywhere

    Skip non-goals, future considerations, and informational sections.

    ## Output Format

    Return EXACTLY this format (one entry per requirement):

    REQUIREMENTS:
    - REQ-1: [source ref, e.g. US-001 or FR-3] | [short description] | CRITERIA: [comma-separated acceptance criteria]
    - REQ-2: ...
    ...

    TOTAL: [count]
```

For **task-based verification** (no PRD), extract requirements from `.loop/tasks.md` instead — each task description becomes a requirement, and completion criteria come from any notes or context in the task.

### Step 3: Verify Each Requirement (Opus)

Spawn an **Opus agent** (verification needs thoroughness) to check the codebase against every extracted requirement:

```
Task tool:
  subagent_type: "code-reviewer"
  model: "opus"
  description: "Verify PRD implementation"
  prompt: |
    ## Project Rules
    [project_rules from config/CLAUDE.md]

    ## Task
    Verify that the codebase correctly implements every requirement listed below.

    ## Requirements to Verify
    [structured checklist from Step 2]

    ## Completed Tasks (for context)
    [completed task list from .loop/tasks.md]

    ## Instructions

    For EACH requirement:
    1. Search the codebase for the relevant implementation (use Glob and Grep)
    2. Read the implementing code
    3. Check each acceptance criterion
    4. Run tests if they exist for this feature
    5. Determine: PASS, FAIL, or PARTIAL

    ## Output Format

    Return EXACTLY this format:

    VERIFICATION REPORT
    ===================

    ## Summary
    Total: [N] requirements
    Pass: [N]
    Partial: [N]
    Fail: [N]
    Coverage: [percent]%

    ## Results

    ### PASS
    - REQ-1: [source ref] | [description] | Evidence: [file:line or test name]
    - REQ-2: ...

    ### PARTIAL (implemented but incomplete)
    - REQ-5: [source ref] | [description] | Missing: [what's incomplete] | Has: [what exists]

    ### FAIL (not implemented)
    - REQ-8: [source ref] | [description] | Expected: [what should exist] | Found: [nothing / wrong impl]

    ### NOTES
    - [any implementation concerns, edge cases, or quality observations]
```

### Step 4: Present Results (Orchestrator)

Display the verification report to the user:

```
Verification: [project name] vs [PRD name]
============================================

Results: 14/16 requirements passed (87%)

PASS (14):
  US-001: Add priority field to database
  US-002: Display priority indicator on task cards
  ...

PARTIAL (1):
  FR-4: Filter tasks by priority
    Has: Filter dropdown in header
    Missing: URL param persistence

FAIL (1):
  FR-5: Sort by priority within columns
    Expected: Priority sorting option in column header
    Found: Not implemented

What would you like to do?
```

### Step 5: Handle Gaps (Orchestrator)

Offer options via AskUserQuestion:

1. **Create tasks for gaps** — Add new tasks to `.loop/tasks.md` for each PARTIAL/FAIL item, then continue the loop
2. **Accept as-is** — Mark the loop as verified with known gaps documented
3. **Re-verify specific items** — Re-check individual requirements after manual fixes

**If "Create tasks for gaps" is selected:**

1. Generate a new task for each PARTIAL and FAIL requirement
2. Append them to `.loop/tasks.md` as a new phase: `### Phase N+1: Verification Fixes`
3. Set dependencies appropriately
4. Log in progress.md: `### [Date] - Verification: [pass]/[total] passed, [N] fix tasks created`
5. Prompt user: "Created [N] fix tasks. Say 'continue loop' to address them."

**If "Accept as-is" is selected:**

1. Log the verification results in progress.md
2. Write a `VERIFICATION.md` report in `.loop/` for the record
3. Mark the loop status as `verified` in tasks.md header

### Verification Report File

When verification completes (regardless of outcome), save the full report:

```markdown
# Verification Report

Project: [name]
Source: [PRD path]
Date: [date]
Result: [pass]/[total] ([percent]%)

## Pass
- [list]

## Partial
- [list with details]

## Fail
- [list with details]

## Notes
- [observations]
```

Save to `.loop/VERIFICATION.md` and commit:
```bash
git add .loop/VERIFICATION.md
git commit -m "chore: add verification report ([pass]/[total] requirements met)"
```

---

## Mode 6: Reset

Start with: "loop reset"

Use this when you want to start fresh on a new set of features without the baggage of a previous project.

### What it does:

1. **Check current state:**
   - Are there incomplete tasks?
   - Are there uncommitted changes?

2. **Confirm with user:**
   ```
   Current loop status:
   - 5/12 tasks completed
   - 7 tasks remaining
   - Last activity: [date]

   Options:
   A. Archive current loop and start fresh
   B. Delete current loop entirely
   C. Cancel
   ```

3. **If archiving:**
   - Move `.loop/` to `.loop-archive/[project-name]-[date]/`
   - Create fresh `.loop/` directory
   - Ready for new `loop plan` or `loop import`

4. **If deleting:**
   - Remove `.loop/` directory
   - Commit the removal
   - Ready for new project

---

## Commands

Users can say:

| Command | Action |
|---------|--------|
| `loop` / `continue loop` | Execute next task(s) |
| `loop plan` | Interactive planning session (recommended for new projects) |
| `loop import [path]` | Generate tasks from PRD or requirements doc |
| `loop from-prd` | Search for and import from recent PRD |
| `loop setup` | Quick manual setup (when you know exact tasks) |
| `loop status` | Show progress summary |
| `loop task [N]` | Execute specific task |
| `loop skip [N]` | Skip a blocked task |
| `loop add [desc]` | Add new task |
| `loop pause` | Stop after current task |
| `loop verify` | Verify implementation against PRD/requirements |
| `loop reset` | Clear current project and start fresh |
| `loop learnings` | Review and promote patterns to CLAUDE.md |
| `loop statusline` | Install a status line showing loop progress |
| `loop statusline off` | Remove the loop status line |

---

## Mode 7: Statusline

Start with: "loop statusline"

Installs a status line at the bottom of the Claude Code TUI showing loop progress, model, and context usage.

### How It Works

This mode uses Claude Code's built-in `/statusline` command to generate a cross-platform bash script. No Node.js, Python, or compiled binaries required -- just bash, which is available everywhere Claude Code runs.

### Install

When the user says "loop statusline", invoke the `/statusline` skill with this prompt:

```
Show a status line with these elements, left to right, separated by dim pipe characters:

1. Model name (dim text) from model.display_name
2. Current loop task: read .loop/tasks.md in the current working directory (workspace.current_dir), find the first line matching "- \[ \]" (pending task) where all dependencies (numbers in "needs: X, Y") correspond to tasks marked "- \[x\]". Show the task description in bold. If no .loop/tasks.md exists or no task is ready, skip this element.
3. Loop progress: count lines matching "- \[x\]" as done and total "- \[" lines as total in .loop/tasks.md. Show as "[done/total]" in dim text. Skip if no .loop/tasks.md.
4. Directory basename (dim text)
5. Context window usage: build a 10-segment progress bar using filled/empty block characters. Color it green below 63%, yellow below 81%, orange below 95%, red+blinking at 95%+. Scale the percentage so that 80% real usage displays as 100% (Claude Code enforces an 80% context limit). Show the scaled percentage number after the bar.

Keep the script portable (bash, no jq, no python, no node). Parse the JSON from stdin using grep/sed/awk. Handle missing .loop/tasks.md gracefully (just skip loop-related elements).
```

After the statusline is generated, confirm to the user:

```
Loop statusline installed! You should see it at the bottom of your screen.

It shows: model | current task | [done/total] | directory | context bar

To remove it later: say "loop statusline off"
```

### Uninstall

When the user says "loop statusline off":

1. Read `~/.claude/settings.json`
2. Remove the `statusLine` key entirely
3. Write the file back
4. Confirm: "Loop statusline removed."

---

## Related Skills

Loop connects to other skills in a pipeline:

```
/gsd:new-project  →  /prd from-gsd  →  loop import  →  continue loop  →  loop verify
 (research)          (consolidate)      (plan tasks)    (execute)         (validate)
```

- **PRD** (`/prd`) — Create requirements documents. Use `/prd` from scratch, or `/prd from-gsd` to convert GSD research into a PRD.
- **GSD** (`/gsd:new-project`) — Deep research and roadmapping. Use before `/prd from-gsd` for thorough upfront planning.
- **Loop** works standalone too — use `loop plan` or `loop setup` when you don't need a PRD.

---

## Status Report

When asked for status, show:

```
Project: [name]
Progress: [done]/[total] tasks ([percent]%)
Models used: [N] sonnet, [N] opus, [N] escalated
Verification: [not run / pass (14/16) / pending]
Source PRD: [path or "none"]

Completed:
  [x] 1. Task one [sonnet]
  [x] 2. Task two [sonnet]

Current/Ready:
  [ ] 3. Task three ← NEXT

Blocked:
  [ ] 4. Task four (waiting on: 3)
  [!] 5. Task five (FAILED on sonnet, escalated to opus - see progress.md)

Remaining: [count] tasks
```

When all tasks are complete, also show:
```
All tasks complete! Next steps:
  loop verify     - Validate implementation against requirements
  loop learnings  - Review and promote patterns to CLAUDE.md
  loop reset      - Archive and start fresh
```

---

## Recovery Scenarios

### Session ended / crashed
Just say "continue loop" - state is in files.

### Task partially complete
Check git status. Either:
- Commit partial progress, update tasks.md manually
- Reset and retry: `git checkout .`

### Wrong task executed
Revert the commit, mark task as pending again, continue.

### Subagent failed or timed out
1. Check what changes were made (git status)
2. Either commit partial progress or reset
3. Mark task as `[!]` with failure reason
4. Continue with next task or retry

### Orchestrator context getting long
Even with subagents, the orchestrator accumulates context from results.
After ~15-20 tasks in one session, recommend fresh start:
```
Orchestrator context is getting long. Recommend starting fresh session.
All state is saved in .loop/ files. Just say "continue loop" to resume.
Progress: [X]/[Y] tasks complete.
```

### Conflicting changes from parallel subagents
If parallel subagents touched the same files:
1. Review the changes manually
2. Resolve conflicts
3. Commit the merged result
4. Mark both tasks complete

---

## Subagent Architecture

Every task runs in a subagent. This ensures fresh context per task, isolation, and parallel execution.

**Orchestrator** (main agent): reads/updates `.loop/` files, spawns subagents, processes results, commits, handles failures.

**Workers** (subagents): read source files, implement the task, run checks, report results. Do NOT commit or modify `.loop/` files.

**Specialized agents:** The full list of 25+ agent types with classification guidance is in `subagents.md` (same directory as this file). The Haiku classifier reads this during Step 3.

**Parallel execution:** If multiple tasks are ready (no dependency conflicts), spawn classification and worker agents in parallel. Wait for all to complete, then update state and commit each.

**Auto-escalation:** Sonnet failure → auto-retry on Opus (if `auto_escalate: true` in config). If Opus also fails → mark `[!]`, ask user.

---

## Knowledge Persistence

The loop uses a two-tier system for learnings:

### Tier 1: Session Learnings (`.loop/progress.md`)

Temporary patterns discovered during this loop execution:
- Specific implementation details
- Workarounds for current task set
- Notes about files touched

Stored in the "Patterns Discovered" section and passed to each subagent.

### Tier 2: Permanent Learnings (`CLAUDE.md`)

Valuable patterns that should persist beyond this loop:
- Project conventions discovered
- API patterns and idioms
- Testing approaches that work
- Architecture decisions made

**Promotion to CLAUDE.md:** If a subagent reports a pattern that is generally applicable (not task-specific), ask the user: "Promote this pattern to CLAUDE.md?" If yes, append under `## Learnings`.

**Automatic capture:** Subagents report patterns as `PATTERN: [category] - [description]`. The orchestrator adds these to progress.md "Patterns Discovered" (always) and offers CLAUDE.md promotion if broadly useful.

**Manual review (`loop learnings`):** Read all patterns from progress.md, present grouped by category, ask which to promote. Useful at end of loop, before `loop reset`, or periodically.

---

## Best Practices

1. **Commit after every task** - Safe checkpoints
2. **Log learnings** - Future tasks benefit from patterns
3. **Promote valuable patterns** - Add lasting knowledge to CLAUDE.md
4. **Keep tasks small** - Easier to complete and recover from
5. **Update progress.md** - Your memory across sessions
6. **Don't skip failures** - Fix or explicitly mark skipped

---

## Checklist

### Before starting execution:
- [ ] Task file exists at `.loop/tasks.md`
- [ ] Progress file exists at `.loop/progress.md`
- [ ] Config file exists at `.loop/config.md`
- [ ] Initial commit made
- [ ] Tasks are small enough (one session each)
- [ ] Dependencies are correctly specified

### After each task:
- [ ] Task marked complete in tasks.md
- [ ] Progress logged in progress.md
- [ ] Changes committed
- [ ] Tests/checks passing

### After all tasks complete:
- [ ] Run `loop verify` to check against PRD/requirements
- [ ] Review PARTIAL and FAIL items
- [ ] Create fix tasks or accept as-is
- [ ] VERIFICATION.md committed

### When starting a new project (existing loop):
- [ ] Run `loop reset` to archive or clear previous project
- [ ] Then use `loop plan` or `loop import`
