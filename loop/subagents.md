# Loop Subagent Reference

This file is read by the Haiku classifier during Step 3 of Mode 4 (Continue).
The orchestrator does NOT need this in its main context.

---

## Available Agent Types

### Language Specialists
Use when the task is primarily in one language.

| Language | subagent_type | Typical model |
|----------|---------------|---------------|
| Go | `golang-pro` | sonnet |
| TypeScript | `typescript-pro` | sonnet |
| JavaScript | `javascript-pro` | sonnet |
| Python | `python-pro` | sonnet |
| Rust | `rust-pro` | sonnet |
| Java | `java-pro` | sonnet |
| SQL | `sql-pro` | sonnet |

### Framework Specialists
Use when the task targets a specific framework.

| Framework | subagent_type | Typical model |
|-----------|---------------|---------------|
| Flutter/Dart | `flutter-expert` | sonnet |
| Serverpod | `serverpod-expert` | sonnet |
| React Native/Flutter | `mobile-developer` | sonnet |
| Terraform/IaC | `terraform-specialist` | sonnet |

### Domain Specialists
Use when the task domain matters more than the language.

| Domain | subagent_type | Typical model |
|--------|---------------|---------------|
| Testing | `test-automator` | sonnet |
| Security/auth | `security-auditor` | **opus** |
| Debugging | `debugger` | **opus** |
| Log/error analysis | `error-detective` | **opus** |
| API design | `backend-architect` | sonnet |
| Architecture review | `architect-reviewer` | sonnet |
| Code review | `code-reviewer` | sonnet |
| CI/CD/Docker | `deployment-engineer` | sonnet |
| Payments/billing | `payment-integration` | **opus** |
| Legacy refactoring | `legacy-modernizer` | sonnet |
| UI components | `frontend-design` | sonnet |
| UX/design | `ui-ux-designer` | sonnet |

### General Purpose

| Use case | subagent_type | Typical model |
|----------|---------------|---------------|
| No specialist match | `general-purpose` | sonnet |
| Context gathering | `general-purpose` | **haiku** |

---

## Classification Guide

### Model Selection

Return **opus** if ANY of these apply:
- Task involves security (auth, encryption, tokens, permissions, input validation)
- Task requires designing new architecture or significant refactoring across 5+ files
- Task involves complex algorithms, concurrency, or race conditions
- Task requires debugging a non-obvious issue
- Task modifies core infrastructure (database schemas, middleware, CI/CD)
- Task involves payment processing or financial logic

Otherwise return **sonnet**.

### Agent Selection

Pick the **most specific** agent that matches:
1. If the task is clearly in one language and that's the main challenge → language specialist
2. If the task targets a specific framework → framework specialist
3. If the task domain is more important than the language (e.g., writing tests, security audit) → domain specialist
4. If nothing fits well → `general-purpose`

The `subagent_type` and `model` are independent — combine them freely.
For example: `subagent_type: "golang-pro", model: "opus"` for a complex Go task.
