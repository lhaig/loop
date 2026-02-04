---
name: loop
description: "Autonomous project execution with crash recovery. Triggers on: loop, start loop, continue loop, run the loop, set up project, resume project, loop plan, loop import. For large features (10+ tasks) with file-based state that persists across sessions."
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

**Four modes:**
1. **Plan** - Interactive planning session that generates the task file
2. **Import** - Generate tasks from an existing PRD or requirements document
3. **Quick Setup** - Manual setup when you already know the tasks
4. **Continue** - Execute tasks via subagents until done or stopped

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

## Subagent Defaults
default_subagent: general-purpose
# Override per phase if needed:
# phase_1_subagent: sql-pro
# phase_3_subagent: test-automator
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

### Example: Importing a PRD

**User:** loop import tasks/prd-priority-system.md

**Assistant:**
```
Reading PRD: Task Priority System

Found 4 user stories:
- US-001: Add priority field to database
- US-002: Display priority indicator on task cards
- US-003: Add priority selector to task edit
- US-004: Filter tasks by priority

Found 5 functional requirements (FR-1 through FR-5)

Proposed task breakdown:

Phase 1: Database
1. Add priority column to tasks table (from US-001)
   - Add migration for priority: 'high' | 'medium' | 'low'
   - Default to 'medium'

Phase 2: UI Components
2. Add priority badge component (from US-002, needs: 1)
   - Colored badges: red/yellow/gray
   - Display on task cards
3. Add priority selector to edit modal (from US-003, needs: 1)

Phase 3: Features
4. Implement priority filtering (from US-004, needs: 2)
   - Filter dropdown in header
   - URL param persistence
5. Add priority sorting within columns (from FR-5, needs: 1)

Shall I create the loop with these 5 tasks?
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

Create `.loop/config.md`:

```markdown
# Loop Config

## Session Limits
max_tasks_per_session: 15
warn_at_tasks: 12

## Project
name: [Project Name]
repo: [current directory]
created: [date]

## Commands
typecheck: [detect from project or ask user]
lint: [detect from project or ask user]
test: [detect from project or ask user]

## Subagent Defaults
default_subagent: general-purpose
```

**Auto-detect commands** by checking:
- `package.json` scripts (npm/node projects)
- `Makefile` targets
- `pyproject.toml` / `setup.py` (Python)
- Ask user if unclear

### Step 7: Commit and Confirm

```bash
mkdir -p .loop
# (write files)
git add .loop/
git commit -m "chore: initialize loop for [project name]"
```

Show the user:
```
Loop initialized!

Project: [name]
Tasks: [count] across [phases] phases

To start: say "continue loop" or "run the loop"
To check status: say "loop status"
```

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

```bash
cat .loop/tasks.md
cat .loop/progress.md
```

Parse:
- Which tasks are done `[x]`
- Which tasks are pending `[ ]`
- Which tasks are blocked `[!]`
- What dependencies are satisfied

### Step 2: Find Next Task (Orchestrator)

Select the first pending task where all dependencies are complete.

If no task is ready:
- All done → Report completion
- Some blocked → Report what's blocking

If multiple tasks are ready:
- Prefer tasks in the same area as recently completed work
- Or ask user which to prioritize

### Step 3: Gather Context (Orchestrator)

Before spawning the subagent, gather everything it needs:

1. **Task details** from tasks.md
2. **Patterns** from progress.md "Patterns Discovered" section
3. **Recent learnings** from last 2-3 task entries in progress.md
4. **Relevant file paths** the task will likely touch
5. **Project conventions** (from CLAUDE.md, AGENTS.md if they exist)

### Step 4: Spawn Subagent (Orchestrator)

Use the Task tool to execute the task:

```
Task tool:
  subagent_type: "general-purpose"
  description: "Task [N]: [short title]"
  prompt: |
    ## Task
    [Full task description from tasks.md]

    ## Context
    Project: [project name]
    Working directory: [path]

    ## Patterns & Learnings
    [Paste relevant patterns from progress.md]

    ## Files Likely Involved
    - [file1]
    - [file2]

    ## Instructions
    1. Read the relevant files first to understand current state
    2. Implement the task
    3. Run type checks / linting: [project-specific command]
    4. Run tests if relevant: [project-specific command]
    5. If checks fail, fix the issues before completing

    ## When Complete
    Report back with:
    - DONE or FAILED
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
```

### Step 5: Process Subagent Result (Orchestrator)

**If DONE:**
1. Update tasks.md: Change `- [ ] N.` to `- [x] N.`
2. Append to progress.md:
   ```markdown
   ### [Date] - Task [N]: [Title]
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

**If FAILED:**
1. Update tasks.md: Change `- [ ] N.` to `- [!] N.`
2. Log failure details in progress.md
3. Ask user: retry, skip, or stop?

**If BLOCKED:**
1. Present the subagent's question to user
2. Get answer
3. Re-spawn subagent with additional context

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
3. If all done → Report completion and summarize

**Auto-continue** until:
- Task fails
- All tasks complete
- User interrupts (Ctrl+C or says "loop pause")
- Subagent reports BLOCKED
- **Session task limit reached**

---

## Mode 5: Reset

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
| `loop reset` | Clear current project and start fresh |
| `loop learnings` | Review and promote patterns to CLAUDE.md |

---

## Status Report

When asked for status, show:

```
Project: [name]
Progress: [done]/[total] tasks ([percent]%)

Completed:
  [x] 1. Task one
  [x] 2. Task two

Current/Ready:
  [ ] 3. Task three ← NEXT

Blocked:
  [ ] 4. Task four (waiting on: 3)
  [!] 5. Task five (FAILED - see progress.md)

Remaining: [count] tasks
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

**Every task runs in a subagent by default.** This ensures:
- Fresh context per task (no buildup over 30+ tasks)
- Isolation (one task's mess doesn't affect others)
- Parallel execution possible (for independent tasks)

### Orchestrator Responsibilities (main agent)
- Read and update `.loop/tasks.md`
- Read and update `.loop/progress.md`
- Gather context for subagents
- Spawn subagents via Task tool
- Process results and commit
- Handle failures and user interaction

### Subagent Responsibilities (worker)
- Read relevant source files
- Implement the task
- Run checks (typecheck, lint, test)
- Report results back
- **NOT** commit (orchestrator does this)
- **NOT** modify .loop/ files (orchestrator does this)

### Specialized Subagents

Use specific subagent types when appropriate:

| Task Type | subagent_type |
|-----------|---------------|
| General implementation | `general-purpose` |
| Writing tests | `test-automator` |
| TypeScript work | `typescript-pro` |
| Security-sensitive | `security-auditor` |
| Database/SQL | `sql-pro` |
| UI components | `frontend-design` |
| Code review | `code-reviewer` |
| Debugging | `debugger` |

### Parallel Execution

If multiple tasks are ready (no dependency conflicts), spawn them in parallel:

```
# In single message, multiple Task tool calls:
Task 1: subagent for task 5
Task 2: subagent for task 6
Task 3: subagent for task 7
```

Wait for all to complete, then update state and commit each.

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

**When to promote to CLAUDE.md:**

After completing a task, if a subagent reports a learning that is:
- Generally applicable (not just for this one task)
- About project conventions or patterns
- Something future work would benefit from

Then the orchestrator should:
1. Ask the user: "This pattern seems valuable. Add to project CLAUDE.md?"
2. If yes, append to `CLAUDE.md` under a `## Learnings` section

**Example promotion:**
```markdown
## Learnings

### Database Patterns
- All migrations use UUID primary keys, not auto-increment
- Foreign keys follow `{table}_id` naming convention

### API Conventions
- All endpoints return `{ data, error, meta }` shape
- Use zod schemas for validation, defined in `src/schemas/`

### Testing
- Integration tests use `setupTestDb()` from `tests/helpers`
- Mock external APIs with `msw` handlers in `tests/mocks/`
```

### Automatic Learning Capture

The subagent prompt includes instructions to report patterns in a structured format:
```
PATTERN: [category] - [description]
```

The orchestrator processes these by:
1. Adding to `.loop/progress.md` "Patterns Discovered" (always)
2. If pattern seems broadly useful, offering to add to `CLAUDE.md`

### Manual Review: `loop learnings`

At any point, run `loop learnings` to:
1. Read all patterns from `.loop/progress.md`
2. Present them grouped by category
3. Ask which to promote to `CLAUDE.md`
4. Update `CLAUDE.md` with selected patterns

This is useful:
- At the end of a loop, to consolidate learnings
- Before `loop reset`, to preserve valuable knowledge
- Periodically during long-running loops

---

## Best Practices

1. **Commit after every task** - Safe checkpoints
2. **Log learnings** - Future tasks benefit from patterns
3. **Promote valuable patterns** - Add lasting knowledge to CLAUDE.md
4. **Keep tasks small** - Easier to complete and recover from
5. **Update progress.md** - Your memory across sessions
6. **Don't skip failures** - Fix or explicitly mark skipped

---

## Example Session

**User:** set up loop for refactoring auth system

**Assistant:** What does the current auth system look like and what's the target state?

**User:** [describes current JWT setup, wants to add refresh tokens and improve security]

**Assistant:** Got it. Here's my proposed breakdown:

Phase 1: Foundation
1. Add refresh_token column to users table
2. Create token generation utilities
3. Update auth middleware

Phase 2: Core
4. Implement refresh token endpoint
5. Update login to return both tokens
6. Add token rotation on refresh

Phase 3: Security
7. Add refresh token revocation
8. Implement token family tracking
9. Add rate limiting

Phase 4: Cleanup
10. Update tests
11. Update API documentation
12. Remove deprecated code

Does this look right?

**User:** yes, create it

**Assistant:** [Creates .loop/tasks.md and .loop/progress.md, commits]

Loop initialized! 12 tasks across 4 phases.
Say "continue loop" to start with task 1.

**User:** continue loop

**Assistant:**
```
Reading state...
- Tasks: 0/12 complete
- Next ready: Task 1 (no dependencies)

Spawning subagent for Task 1: Add refresh_token column
```
[Spawns subagent via Task tool]

```
Subagent result: DONE
- Files changed: db/migrations/20260125_add_refresh_token.sql, db/schema.ts
- Learnings: Using nullable column for backward compatibility

Committing: feat(auth): add refresh_token column to users table
Task 1 complete (1/12)

Next ready: Tasks 2, 3 (both depend only on task 1)
Spawning subagents in parallel...
```
[Spawns 2 subagents in parallel]

```
Task 2: DONE - Created tokenUtils.ts with generate/verify functions
Task 3: DONE - Updated middleware to check token expiry

Committing both...
Tasks 2, 3 complete (3/12)

Next ready: Task 4 (depends on 2, 3 - both done)
Spawning subagent...
```

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

### When starting a new project (existing loop):
- [ ] Run `loop reset` to archive or clear previous project
- [ ] Then use `loop plan` or `loop import`
