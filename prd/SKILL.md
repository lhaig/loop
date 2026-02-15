---
name: prd
description: "Generate a Product Requirements Document (PRD) for a new feature. Use when planning a feature, starting a new project, or when asked to create a PRD. Triggers on: create a prd, write prd for, plan this feature, requirements for, spec out."
---

# PRD Generator

Create detailed Product Requirements Documents that are clear, actionable, and suitable for implementation.

## The Job

1. Receive a feature description from the user
2. Ask 3-5 essential clarifying questions (with lettered options)
3. Generate a structured PRD based on answers
4. Save to `.planning/prd-[feature-name].md`

**Important:** Do NOT start implementing. Just create the PRD.

---

## Related Skills

```
/prd              →  loop import  →  continue loop  →  loop verify
 (requirements)      (plan tasks)    (execute)         (validate)
```

- **Loop** — Autonomous task execution. Use `loop import .planning/prd-*.md` to import a PRD as executable tasks.

---

## Step 1: Clarifying Questions

Ask only critical questions where the initial prompt is ambiguous. Focus on:

- **Problem/Goal:** What problem does this solve?
- **Core Functionality:** What are the key actions?
- **Scope/Boundaries:** What should it NOT do?
- **Success Criteria:** How do we know it's done?

Format with lettered options so users can respond quickly (e.g. "1A, 2C, 3B"):

```
1. What is the primary goal?
   A. Option one
   B. Option two
   C. Other: [please specify]
```

---

## Step 2: PRD Structure

Generate the PRD with these sections:

### 1. Introduction/Overview
Brief description of the feature and the problem it solves.

### 2. Goals
Specific, measurable objectives (bullet list).

### 3. User Stories
Each story should be small enough to implement in one focused session.

```markdown
### US-001: [Title]
**Description:** As a [user], I want [feature] so that [benefit].

**Acceptance Criteria:**
- [ ] Specific verifiable criterion
- [ ] Another criterion
```

**Important:** Acceptance criteria must be verifiable, not vague. "Works correctly" is bad. "Button shows confirmation dialog before deleting" is good.

### 4. Functional Requirements
Numbered list: "FR-1: The system must..."

### 5. Non-Goals (Out of Scope)
What this feature will NOT include.

### 6. Technical Considerations (Optional)
Known constraints, dependencies, integration points, performance requirements.

### 7. Success Metrics
How will success be measured?

### 8. Open Questions
Remaining questions or areas needing clarification.

---

## Writing Style

The PRD reader may be a junior developer or AI agent. Therefore:
- Be explicit and unambiguous
- Avoid jargon or explain it
- Number requirements for easy reference
- Use concrete examples where helpful

---

## Output

- **Format:** Markdown (`.md`)
- **Location:** `.planning/`
- **Filename:** `prd-[feature-name].md` (kebab-case)

---

## Example PRD

```markdown
# PRD: Task Priority System

## Introduction
Add priority levels to tasks so users can focus on what matters most.

## Goals
- Allow assigning priority (high/medium/low) to any task
- Provide clear visual differentiation between priority levels
- Enable filtering and sorting by priority

## User Stories

### US-001: Add priority field to database
**Description:** As a developer, I need to store task priority so it persists across sessions.

**Acceptance Criteria:**
- [ ] Add priority column to tasks table: 'high' | 'medium' | 'low' (default 'medium')
- [ ] Generate and run migration successfully

### US-002: Display priority indicator on task cards
**Description:** As a user, I want to see task priority at a glance.

**Acceptance Criteria:**
- [ ] Each task card shows colored priority badge
- [ ] Priority visible without hovering or clicking

### US-003: Filter tasks by priority
**Description:** As a user, I want to filter the task list to see only high-priority items.

**Acceptance Criteria:**
- [ ] Filter dropdown with options: All | High | Medium | Low
- [ ] Empty state message when no tasks match filter

## Functional Requirements
- FR-1: Add `priority` field to tasks table (default 'medium')
- FR-2: Display colored priority badge on each task card
- FR-3: Add priority filter dropdown to task list header

## Non-Goals
- No priority-based notifications
- No automatic priority assignment
- No priority inheritance for subtasks

## Technical Considerations
- Reuse existing badge component with color variants
- Filter state managed via URL search params

## Success Metrics
- Users can change priority in <2 clicks
- No regression in task list performance

## Open Questions
- Should priority affect task ordering within a column?
```

---

## Checklist

- [ ] Asked clarifying questions with lettered options
- [ ] Incorporated user's answers
- [ ] User stories are small and specific
- [ ] Functional requirements are numbered and unambiguous
- [ ] Non-goals section defines clear boundaries
- [ ] Saved to `.planning/prd-[feature-name].md`
