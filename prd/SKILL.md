---
name: prd
description: "Generate a Product Requirements Document (PRD) for a new feature. Use when planning a feature, starting a new project, or when asked to create a PRD. Triggers on: create a prd, write prd for, plan this feature, requirements for, spec out, prd from-gsd, from-gsd."
---

# PRD Generator

Create detailed Product Requirements Documents that are clear, actionable, and suitable for implementation.

---

## The Job

**Standard mode** (from scratch):
1. Receive a feature description from the user
2. Ask 3-5 essential clarifying questions (with lettered options)
3. Generate a structured PRD based on answers
4. Save to `/tasks/prd-[feature-name].md`

**From GSD mode** (`/prd from-gsd`):
1. Read GSD artifacts from `.planning/` directory
2. Map GSD structures to PRD format
3. Present draft for review
4. Save to `/tasks/prd-[feature-name].md`

**Important:** Do NOT start implementing. Just create the PRD.

---

## Commands

| Command | Action |
|---------|--------|
| `/prd` | Create a PRD from scratch (interactive Q&A) |
| `/prd from-gsd` | Generate PRD from GSD `.planning/` artifacts |

---

## Related Skills

The PRD connects to other skills in a pipeline:

```
/gsd:new-project  →  /prd from-gsd  →  loop import  →  continue loop  →  loop verify
 (research)          (consolidate)      (plan tasks)    (execute)         (validate)
```

- **GSD** — Deep research, requirements gathering, and roadmapping. Use `/gsd:new-project` first, then `/prd from-gsd` to consolidate.
- **Loop** — Autonomous task execution. Use `loop import tasks/prd-*.md` to import a PRD as executable tasks.

---

## Step 1: Clarifying Questions

Ask only critical questions where the initial prompt is ambiguous. Focus on:

- **Problem/Goal:** What problem does this solve?
- **Core Functionality:** What are the key actions?
- **Scope/Boundaries:** What should it NOT do?
- **Success Criteria:** How do we know it's done?

### Format Questions Like This:

```
1. What is the primary goal of this feature?
   A. Improve user onboarding experience
   B. Increase user retention
   C. Reduce support burden
   D. Other: [please specify]

2. Who is the target user?
   A. New users only
   B. Existing users only
   C. All users
   D. Admin users only

3. What is the scope?
   A. Minimal viable version
   B. Full-featured implementation
   C. Just the backend/API
   D. Just the UI
```

This lets users respond with "1A, 2C, 3B" for quick iteration.

---

## Step 2: PRD Structure

Generate the PRD with these sections:

### 1. Introduction/Overview
Brief description of the feature and the problem it solves.

### 2. Goals
Specific, measurable objectives (bullet list).

### 3. User Stories
Each story needs:
- **Title:** Short descriptive name
- **Description:** "As a [user], I want [feature] so that [benefit]"
- **Acceptance Criteria:** Verifiable checklist of what "done" means

Each story should be small enough to implement in one focused session.

**Format:**
```markdown
### US-001: [Title]
**Description:** As a [user], I want [feature] so that [benefit].

**Acceptance Criteria:**
- [ ] Specific verifiable criterion
- [ ] Another criterion
- [ ] npm run typecheck passes
- [ ] **[UI stories only]** Verify in browser using dev-browser skill
```

**Important:** 
- Acceptance criteria must be verifiable, not vague. "Works correctly" is bad. "Button shows confirmation dialog before deleting" is good.
- **For any story with UI changes:** Always include "Verify in browser using dev-browser skill" as acceptance criteria. This ensures visual verification of frontend work.

### 4. Functional Requirements
Numbered list of specific functionalities:
- "FR-1: The system must allow users to..."
- "FR-2: When a user clicks X, the system must..."

Be explicit and unambiguous.

### 5. Non-Goals (Out of Scope)
What this feature will NOT include. Critical for managing scope.

### 6. Design Considerations (Optional)
- UI/UX requirements
- Link to mockups if available
- Relevant existing components to reuse

### 7. Technical Considerations (Optional)
- Known constraints or dependencies
- Integration points with existing systems
- Performance requirements

### 8. Success Metrics
How will success be measured?
- "Reduce time to complete X by 50%"
- "Increase conversion rate by 10%"

### 9. Open Questions
Remaining questions or areas needing clarification.

---

## Mode: From GSD

Start with: `/prd from-gsd` or `prd from-gsd`

This mode reads GSD's `.planning/` artifacts and consolidates them into a standard PRD. Useful when you've done thorough research and planning in GSD but want to execute via the loop skill.

### When to Use

- After running `/gsd:new-project` (research + requirements + roadmap complete)
- When `.planning/PROJECT.md` and `.planning/REQUIREMENTS.md` exist
- Before `loop import` to create an executable task source

### Step 1: Detect and Read GSD Artifacts

Check for `.planning/` directory and read all available files:

**Required files** (abort if missing):
- `.planning/PROJECT.md` — project context, core value, constraints
- `.planning/REQUIREMENTS.md` — categorized requirements with traceability

**Optional files** (enrich the PRD if present):
- `.planning/ROADMAP.md` — phases, goals, dependencies, success criteria
- `.planning/phases/*/XX-RESEARCH.md` — library decisions, architecture patterns
- `.planning/phases/*/DISCOVERY.md` — shallow research, recommendations
- `.planning/phases/*/XX-CONTEXT.md` — implementation decisions per phase
- `.planning/phases/*/XX-XX-PLAN.md` — detailed task breakdowns

If `.planning/` doesn't exist or required files are missing:
```
No GSD artifacts found in .planning/

To use from-gsd mode, first run /gsd:new-project to create:
  - .planning/PROJECT.md
  - .planning/REQUIREMENTS.md
  - .planning/ROADMAP.md

Or use /prd without from-gsd to create a PRD from scratch.
```

### Step 2: Map GSD to PRD Structure

Read each file and map to PRD sections:

| GSD Source | PRD Section | How to Map |
|------------|-------------|------------|
| `PROJECT.md` "What This Is" + "Core Value" | **Introduction/Overview** | Combine into 2-3 sentence overview |
| `PROJECT.md` constraints + `ROADMAP.md` phase goals | **Goals** | Extract measurable objectives from roadmap goals and success criteria |
| `REQUIREMENTS.md` requirements | **Functional Requirements** | Convert `AUTH-01: description` → `FR-1: description`, preserve categories as grouping headers |
| `REQUIREMENTS.md` + `ROADMAP.md` success criteria | **User Stories** | Transform each requirement into US-xxx format with acceptance criteria derived from success criteria |
| `REQUIREMENTS.md` "Out of Scope" + `PROJECT.md` out of scope | **Non-Goals** | Merge and deduplicate, keep the reasoning |
| `RESEARCH.md` standard stack + architecture patterns | **Technical Considerations** | Summarize library choices, architecture decisions, and constraints |
| `DISCOVERY.md` recommendations | **Technical Considerations** | Add specific library recommendations with versions |
| `ROADMAP.md` phases | **User Story ordering** | Phase sequence determines story dependencies |
| `CONTEXT.md` implementation decisions | **Design Considerations** | Include locked decisions and UI/UX choices |

### Step 3: Generate User Stories from Requirements

This is the critical transformation. GSD requirements are terse; PRD user stories need structure.

For each requirement in `REQUIREMENTS.md`:

1. **Create a user story ID**: Number sequentially (US-001, US-002, etc.)
2. **Write the description**: Transform `AUTH-01: User can sign up with email and password` into `As a new user, I want to sign up with email and password so that I can access the application.`
3. **Derive acceptance criteria** from:
   - The requirement itself (the core criterion)
   - `ROADMAP.md` success criteria for the phase that covers this requirement
   - `CONTEXT.md` implementation decisions that affect this requirement
   - Standard quality checks (typecheck, lint, tests)
   - UI verification where applicable

**Example transformation:**
```
GSD REQUIREMENTS.md:
  AUTH-01: User can sign up with email and password
  (Phase 1, Success Criteria: "User completes signup in under 30 seconds")

GSD CONTEXT.md:
  - Email validation: RFC 5322 format check
  - Password: minimum 8 characters, at least one number

PRD User Story:
  ### US-001: User signup with email and password
  **Description:** As a new user, I want to sign up with email and password
  so that I can create an account and access the application.

  **Acceptance Criteria:**
  - [ ] Signup form with email and password fields
  - [ ] Email validated against RFC 5322 format
  - [ ] Password requires minimum 8 characters with at least one number
  - [ ] Successful signup creates user record in database
  - [ ] Error messages shown for invalid input
  - [ ] Signup completable in under 30 seconds
  - [ ] Tests pass
  - [ ] Verify in browser using dev-browser skill
```

### Step 4: Add Phase Annotations

Since GSD organizes work into phases and the loop skill uses phases too, preserve this structure by annotating each user story with its source phase:

```markdown
### US-001: User signup with email and password
<!-- GSD: AUTH-01, Phase 1: Authentication -->
**Description:** ...
```

This helps `loop import` maintain the same phase grouping.

### Step 5: Build Technical Considerations

Consolidate research into a concise section:

```markdown
## Technical Considerations

### Library Decisions (from GSD Research)
| Library | Version | Purpose |
|---------|---------|---------|
| bcrypt  | 5.1.1   | Password hashing |
| zod     | 3.22    | Input validation |

### Architecture Decisions
- [From CONTEXT.md: locked decisions]
- [From RESEARCH.md: architecture patterns chosen]

### Constraints
- [From PROJECT.md: hard constraints]

### Known Risks
- [From RESEARCH.md: common pitfalls section]
```

### Step 6: Present for Review

Show the user the generated PRD before saving:

```
Generated PRD from GSD artifacts:

Source: .planning/ (PROJECT.md, REQUIREMENTS.md, ROADMAP.md, 2 RESEARCH files)

PRD Summary:
  - 12 user stories across 4 phases
  - 15 functional requirements
  - 5 non-goals
  - 8 library decisions documented

Sections:
  1. Introduction (from PROJECT.md)
  2. Goals (4 objectives from ROADMAP.md)
  3. User Stories (12 stories from REQUIREMENTS.md)
  4. Functional Requirements (15 from REQUIREMENTS.md)
  5. Non-Goals (5 from PROJECT.md + REQUIREMENTS.md)
  6. Design Considerations (from CONTEXT.md)
  7. Technical Considerations (from RESEARCH.md)
  8. Success Metrics (from ROADMAP.md success criteria)
  9. Open Questions (from PROJECT.md + RESEARCH.md)

Ready to save to /tasks/prd-[name].md?
Or would you like to review/modify any section first?
```

### Step 7: Save and Suggest Next Steps

Save the PRD and suggest the loop workflow:

```
Saved: /tasks/prd-[feature-name].md

GSD artifacts preserved in .planning/ (not modified).

Next steps:
  loop import tasks/prd-[feature-name].md   # Import as executable tasks
  continue loop                              # Start execution
  loop verify                                # Validate when done
```

### Handling Partial GSD Artifacts

Not every GSD project will have all files. Handle gracefully:

| Missing File | Impact | Fallback |
|--------------|--------|----------|
| `ROADMAP.md` | No phase structure or success criteria | Group stories by requirement category, ask user for success criteria |
| `RESEARCH.md` | No technical considerations | Skip section or mark as "TBD - research recommended" |
| `CONTEXT.md` | No implementation decisions | Derive from REQUIREMENTS.md constraints only |
| `PLAN.md` files | No detailed task breakdowns | User stories stay higher-level; loop will handle breakdown |
| `DISCOVERY.md` | No library recommendations | Skip or note "No library decisions documented" |

### Example: Full GSD → PRD Conversion

**GSD Input:**
```
.planning/
  PROJECT.md          → "Task management app with priority system"
  REQUIREMENTS.md     → AUTH-01, AUTH-02, TASK-01..04, PRI-01..03
  ROADMAP.md          → 3 phases: Auth, Tasks, Priority
  phases/
    01-auth/
      01-auth-RESEARCH.md    → bcrypt, JWT, session management
      01-auth-CONTEXT.md     → "Use HTTP-only cookies, not localStorage"
    02-tasks/
      02-tasks-CONTEXT.md    → "CRUD with soft delete"
    03-priority/
      03-priority-CONTEXT.md → "Three levels: high/medium/low"
```

**PRD Output:**
```markdown
# PRD: Task Management with Priority System

## Introduction
A task management application that helps users organize work with
priority levels. Users can create, manage, and prioritize tasks
with clear visual indicators and filtering.

## Goals
- Secure user authentication with session management
- Full task CRUD with soft delete for recovery
- Priority system (high/medium/low) with visual differentiation
- Filter and sort tasks by priority

## User Stories

### US-001: User signup
<!-- GSD: AUTH-01, Phase 1: Authentication -->
**Description:** As a new user, I want to create an account...

**Acceptance Criteria:**
- [ ] Signup form with email/password
- [ ] Password hashed with bcrypt
- [ ] Session created via HTTP-only cookie (not localStorage)
- [ ] Tests pass
- [ ] Verify in browser using dev-browser skill

[... more stories ...]

### US-007: Assign priority to task
<!-- GSD: PRI-01, Phase 3: Priority -->
**Description:** As a user, I want to set task priority...

**Acceptance Criteria:**
- [ ] Priority selector with high/medium/low options
- [ ] Default to medium for new tasks
- [ ] Visual badge: colored indicator per level
- [ ] Tests pass
- [ ] Verify in browser using dev-browser skill

## Functional Requirements
### Authentication
- FR-1: User signup with email and password (AUTH-01)
- FR-2: User login with session management (AUTH-02)

### Tasks
- FR-3: Create task with title and description (TASK-01)
- FR-4: Edit existing tasks (TASK-02)
- FR-5: Soft delete tasks with recovery option (TASK-03)
- FR-6: List tasks with pagination (TASK-04)

### Priority
- FR-7: Assign priority level to tasks (PRI-01)
- FR-8: Display priority badges on task cards (PRI-02)
- FR-9: Filter task list by priority (PRI-03)

## Non-Goals
- No real-time collaboration
- No mobile app (web only)
- No task assignment to other users

## Technical Considerations

### Library Decisions
| Library | Version | Purpose |
|---------|---------|---------|
| bcrypt  | 5.1.1   | Password hashing |
| ...     | ...     | ... |

### Architecture Decisions
- HTTP-only cookies for session management (not localStorage)
- Soft delete pattern for tasks (deleted_at column)
- Three-level priority enum in database

## Success Metrics
- Signup to first task created in under 2 minutes
- Priority change in under 2 clicks
- Page load under 200ms

## Open Questions
- Should priority affect default sort order?
```

---

## Writing for Junior Developers

The PRD reader may be a junior developer or AI agent. Therefore:

- Be explicit and unambiguous
- Avoid jargon or explain it
- Provide enough detail to understand purpose and core logic
- Number requirements for easy reference
- Use concrete examples where helpful

---

## Output

- **Format:** Markdown (`.md`)
- **Location:** `/tasks/`
- **Filename:** `prd-[feature-name].md` (kebab-case)

---

## Example PRD

```markdown
# PRD: Task Priority System

## Introduction

Add priority levels to tasks so users can focus on what matters most. Tasks can be marked as high, medium, or low priority, with visual indicators and filtering to help users manage their workload effectively.

## Goals

- Allow assigning priority (high/medium/low) to any task
- Provide clear visual differentiation between priority levels
- Enable filtering and sorting by priority
- Default new tasks to medium priority

## User Stories

### US-001: Add priority field to database
**Description:** As a developer, I need to store task priority so it persists across sessions.

**Acceptance Criteria:**
- [ ] Add priority column to tasks table: 'high' | 'medium' | 'low' (default 'medium')
- [ ] Generate and run migration successfully
- [ ] npm run typecheck passes

### US-002: Display priority indicator on task cards
**Description:** As a user, I want to see task priority at a glance so I know what needs attention first.

**Acceptance Criteria:**
- [ ] Each task card shows colored priority badge (red=high, yellow=medium, gray=low)
- [ ] Badge includes icon: 🔴 high, 🟡 medium, ⚪ low
- [ ] Priority visible without hovering or clicking
- [ ] npm run typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-003: Add priority selector to task edit
**Description:** As a user, I want to change a task's priority when editing it.

**Acceptance Criteria:**
- [ ] Priority dropdown in task edit modal
- [ ] Shows current priority as selected
- [ ] Saves immediately on selection change
- [ ] npm run typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-004: Filter tasks by priority
**Description:** As a user, I want to filter the task list to see only high-priority items when I'm focused.

**Acceptance Criteria:**
- [ ] Filter dropdown with options: All | High | Medium | Low
- [ ] Filter persists in URL params
- [ ] Empty state message when no tasks match filter
- [ ] npm run typecheck passes
- [ ] Verify in browser using dev-browser skill

## Functional Requirements

- FR-1: Add `priority` field to tasks table ('high' | 'medium' | 'low', default 'medium')
- FR-2: Display colored priority badge on each task card
- FR-3: Include priority selector in task edit modal
- FR-4: Add priority filter dropdown to task list header
- FR-5: Sort by priority within each status column (high → medium → low)

## Non-Goals

- No priority-based notifications or reminders
- No automatic priority assignment based on due date
- No priority inheritance for subtasks

## Technical Considerations

- Reuse existing badge component with color variants
- Filter state managed via URL search params
- Priority stored in database, not computed

## Success Metrics

- Users can change priority in <2 clicks
- High-priority tasks immediately visible at top of lists
- No regression in task list performance

## Open Questions

- Should priority affect task ordering within a column?
- Should we add keyboard shortcuts for priority changes?
```

---

## Checklist

### Standard mode:
- [ ] Asked clarifying questions with lettered options
- [ ] Incorporated user's answers
- [ ] User stories are small and specific
- [ ] Functional requirements are numbered and unambiguous
- [ ] Non-goals section defines clear boundaries
- [ ] Saved to `/tasks/prd-[feature-name].md`

### From GSD mode:
- [ ] Read all available `.planning/` files
- [ ] PROJECT.md and REQUIREMENTS.md both present
- [ ] Every requirement in REQUIREMENTS.md has a corresponding user story
- [ ] Acceptance criteria derived from success criteria + CONTEXT.md decisions
- [ ] Phase annotations (GSD comments) preserved on user stories
- [ ] Technical considerations include library decisions from RESEARCH.md
- [ ] Non-goals merged from both PROJECT.md and REQUIREMENTS.md
- [ ] Presented summary to user for review before saving
- [ ] Saved to `/tasks/prd-[feature-name].md`
- [ ] Suggested `loop import` as next step
