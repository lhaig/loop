# Norman Subagent Reference

This file is read by the Haiku classifier during Step 3 of Mode 4 (Continue).
The orchestrator does NOT need this in its main context.

---

## Advisor Pattern

Norman uses a three-tier advisor model:

| Role | Model | Purpose |
|------|-------|---------|
| **Advisor** | Opus | Reviews plans before execution, reviews completed code, diagnoses failures |
| **Worker** | Sonnet | Implements all tasks (always sonnet, never opus) |
| **Support** | Haiku | Classifies tasks, gathers context, compresses progress |

Workers are always Sonnet. The Opus advisor handles quality control through plan review (Step 3.1) and code review (Step 6.1). This separation means Opus spends tokens on reasoning about approach and quality rather than writing boilerplate.

---

## Available Agent Types

### Language Specialists
Use when the task is primarily in one language.

| Language | subagent_type |
|----------|---------------|
| Go | `golang-pro` |
| TypeScript | `typescript-pro` |
| JavaScript | `javascript-pro` |
| Python | `python-pro` |
| Rust | `rust-pro` |
| Java | `java-pro` |
| SQL | `sql-pro` |

### Framework Specialists
Use when the task targets a specific framework.

| Framework | subagent_type |
|-----------|---------------|
| Flutter/Dart | `flutter-expert` |
| Serverpod | `serverpod-expert` |
| React Native/Flutter | `mobile-developer` |
| Terraform/IaC | `terraform-specialist` |

### Domain Specialists
Use when the task domain matters more than the language.

| Domain | subagent_type |
|--------|---------------|
| Testing | `test-automator` |
| Security/auth | `security-auditor` |
| Debugging | `debugger` |
| Log/error analysis | `error-detective` |
| API design | `backend-architect` |
| Architecture review | `architect-reviewer` |
| Code review | `code-reviewer` |
| CI/CD/Docker | `deployment-engineer` |
| Payments/billing | `payment-integration` |
| Legacy refactoring | `legacy-modernizer` |
| UI components | `frontend-design` |
| UX/design | `ui-ux-designer` |

### General Purpose

| Use case | subagent_type |
|----------|---------------|
| No specialist match | `general-purpose` |
| Context gathering (haiku) | `general-purpose` |

---

## Classification Guide

### Complexity Classification

The classifier returns a `COMPLEXITY` level used to decide whether the Opus advisor reviews the task (in `auto` mode):

Return **high** if ANY of these apply:
- Task involves security (auth, encryption, tokens, permissions, input validation)
- Task requires designing new architecture or significant refactoring across 5+ files
- Task involves complex algorithms, concurrency, or race conditions
- Task requires debugging a non-obvious issue
- Task modifies core infrastructure (database schemas, middleware, CI/CD)
- Task involves payment processing or financial logic

Return **medium** if ANY of these apply:
- Task touches 3-4 files
- Task involves non-trivial business logic
- Task has integration points with external services
- Task modifies shared utilities or common code

Otherwise return **low**.

In `always` mode, the advisor reviews every task regardless of complexity. In `never` mode, the advisor is skipped entirely.

### Agent Selection

Pick the **most specific** agent that matches:
1. If the task is clearly in one language and that's the main challenge -> language specialist
2. If the task targets a specific framework -> framework specialist
3. If the task domain is more important than the language (e.g., writing tests, security audit) -> domain specialist
4. If nothing fits well -> `general-purpose`

All workers run on Sonnet. The agent type determines the specialist prompt, not the model.
