---
name: codereview
description: >-
  Stage 7 of the agent pipeline. Convenes a multi-reviewer specialist panel
  (Principal Architect, security lenses, correctness auditors) to review staged or committed
  diffs against architectural rules and safety invariants. Use before pushing or merging code.
---

# Code Review

**Invoke via:** `/codereview`

**You are the coordinator.** Execute Steps 1–8 in order yourself. Main agent owns the deliverable — see [Agent Pipeline Contract](../agent-pipeline/SKILL.md).

Spawn a **Review Panel** of specialized subagents in Step 5 (`readonly: true`). You merge findings and run verification gates in **this** conversation.

**Requires:** Staged or committed changes from `/execute-plan` (or user-provided diff scope).

**Pipeline:** See [Agent Pipeline Contract](../agent-pipeline/SKILL.md). Plan-level gaps → `/update-plan` or `/converge-plan`. Clean → `/open-mr`.

## Execution checklist

| Step | Who | Action | Gate |
| :--- | :--- | :--- | :--- |
| 1 | Main agent | **Context recall** (`AGENTS.md`, plan, execution outcome) | Prior context loaded |
| 2 | Main agent | Scan diff + `@ai-shebang` on touched files | Modified file list |
| 3 | Main agent | Build **Code Review Context Package** | Plan + diff paths attached |
| 4 | Main agent | **Review Panel triage** + read applicable skills | Panel table published |
| 5 | Review panel (parallel) | Dispatch specialized reviewers (`readonly: true`) | All returned |
| 6 | Main agent | Pessimistic merge + correlation | Unified report draft |
| 7 | Main agent | Verification gate (test, typecheck, build) | Pass or failures listed |
| 8 | Main agent | Write report & present in chat | User sees merged review + CTA |

**Next (if clean):** `/open-mr` (push, create PR/MR, monitor CI & reviews, merge when safe), then `/eop`.

---

## Step 1: Context Recall

Read `AGENTS.md` (workspace and user-level if present), the active `.plan.md`, and the latest `*-execution-outcome_*.md` if available.

---

## Step 2: Diff Scope

1. Identify all modified files (`git diff --staged` or branch diff `git diff <base>...HEAD`).
2. Read `@ai-shebang` on every touched file.
3. List consumers/callers of modified exports (initial pass).
4. Estimate diff size (lines changed) and domains touched (API, auth, DB, UI, platform, etc.).
5. **Security surface scan** — flag if diff touches: auth/authz, secrets/env, user input parsing, public endpoints, crypto/TLS, IAM/RBAC, SQL/command construction, file/path handling, dependency manifests (`package.json`, `go.mod`, `requirements.txt`), container/K8s security context, logging of sensitive data.

---

## Step 3: Code Review Context Package (by Reference)

**Main agent only.** Read the original `.plan.md` from disk if available and build the package by reference:

```markdown
## Code Review Context Package: {plan-slug}

### 1. Plan Identity
- Plan path (read via file tools), overview, workspace
- Pre-flight report path/summary if available

### 2. Execution Scope
- Which plan todos were implemented
- Commits or summary of changes

### 3. Diff Scope (by Reference)
- Git range: `git diff <base>...HEAD`
- Modified files (repo-relative paths) and line counts
- Hexagonal layer per file (Domain / Port / Adapter / Infrastructure)

### 4. Consumer Map (initial)
- Callers/consumers from Step 2

### 5. Prior Art
- Relevant `AGENTS.md` constraints from Step 1

### 6. Review Mission
Review changes against plan intent. System-wide impact, contracts, zero deferred debt. Do not rubber-stamp.
```

**Gate:** Sections 1–3 complete before Step 5.

---

## Step 4: Review Panel Triage

**Main agent only.** Select reviewers from the tables below. Publish the panel in chat before dispatching.

### Skills (read before Step 5 — inject criteria into reviewer prompts)

| Skill path | When to read | Attach checklist to |
| :--- | :--- | :--- |
| `skills/platform-design-review/SKILL.md` | APIs, IaC, platform abstractions, SDK/CLI surface | `codereview`, `ce-architecture-strategist` |

### Core panel (always dispatch)

| ID | `subagent_type` | Lens |
| :--- | :--- | :--- |
| **R1** | `codereview` | Principal review — architecture, contracts, downstream impact (`agents/codereview.md`) |
| **R2** | `ce-correctness-reviewer` | Logic, edge cases, state bugs |
| **R3** | `ce-maintainability-reviewer` | Structure, coupling, naming, complexity |
| **R4** | `code-reviewer` | **Plan alignment** — implementation vs plan todos & evidence (skip only if no plan) |
| **R5** | `ce-security-reviewer` | **Exploitable diff security** — auth/authz, injection, input validation, IDOR, SSRF, path traversal in changed code |
| **R6** | `ce-security-sentinel` | **Security audit** — secrets/credentials in diff, OWASP patterns, unsafe crypto, dependency CVE surface, sensitive data in logs |

**Security core is mandatory** for every `/codereview` after `/execute-plan`. Only skip R5+R6 if the diff is **docs-only** (markdown, comments, no executable/config-IaC changes) — state that explicitly in the triage table.

### Conditional panel (dispatch when signal matches)

| ID | `subagent_type` | Trigger |
| :--- | :--- | :--- |
| **C1** | `ce-testing-reviewer` | **Mandatory after `/execute-plan`** (TDD gate). Also triggers on: tests changed or production logic without test updates |
| **C2** | `ce-api-contract-reviewer` | API routes, types, serialization, exported signatures |
| **C4** | `ce-reliability-reviewer` | Retries, timeouts, jobs, error handling, async handlers |
| **C5** | `ce-performance-reviewer` | Queries, loops, caching, I/O-heavy paths |
| **C6** | `ce-adversarial-reviewer` | Diff ≥ 50 lines OR auth/payments/data-mutation paths |
| **C7** | `ce-data-integrity-guardian` | Migrations, models, persistent data constraints |
| **C8** | `ce-data-migration-reviewer` | Migration files, schema dumps, backfills |
| **C9** | `ce-julik-frontend-races-reviewer` | Async UI, DOM timing, frontend state races |
| **C10** | `ce-swift-ios-reviewer` | Swift, SwiftUI, iOS project files |
| **C11** | `ce-architecture-strategist` | New services, large refactors, cross-boundary changes |
| **C12** | `ce-project-standards-reviewer` | Any change — project `AGENTS.md` / conventions |
| **C13** | `ce-pattern-recognition-specialist` | New abstractions or inconsistent patterns suspected |

**Panel limits:** Dispatch **all core** (R1–R6 minus R4 if no plan; minus R5+R6 only for docs-only diffs) **plus** matching conditional reviewers in **one message**. Cap at **12 parallel** — if more match, prioritize: **R5/R6** (never drop) → C6 → C2 → C7/C8 → C1 → remaining.

**Security merge rule:** Any **HIGH** security finding from R5, R6, or C6 → overall risk **cannot** be Low; blocks push until fixed or explicitly accepted by user.

**Model policy:** Thinking-tier only. No `*-fast` models.

---

## Step 5: Parallel Reviews

**WAIT** for all panel members before Step 6. Dispatch in parallel with `readonly: true`.

Each prompt MUST include:
1. **Code Review Context Package by Reference**
2. **Communication Contract:** strict telegraphic style (caveman), zero conversational filler
3. **Reviewer role and lens**
4. **Mission:** return structured findings only
5. **Signature:** reviewer signs findings with their ID (R1, R5, etc.)

---

## Step 6: Merge (Pessimistic)

| Rule | Policy |
| :--- | :--- |
| Risk level | **Highest** across reviewers |
| Cynefin | **More complex** wins |
| Findings | **Union** all — tag with reviewer ID |
| Severity | **Highest** wins |
| Downstream impact | **Union** consumers |
| Security findings | Any exploitable issue = **HIGH** unless proven false positive |

Include dedicated **Security section** and **Reviewer agreement matrix**.

---

## Step 7: Verification Gate

| Check | Command | Result |
| :--- | :--- | :--- |
| Unit tests | project test command | pass/fail |
| Typecheck | tsc/mypy/etc. | pass/fail |
| Build | build command | pass/fail |
| Test spec coverage | Compare test files vs plan Step 7a table | all covered / gaps listed |
| TDD divergence | Review reconciliation summary | resolved / unresolved |

Do NOT approve push if any check fails.

---

## Step 8: Write & Present

Path: `.agents/plans/{plan-slug}-Code_Review_YYYY-MM-DD_HHMM.md` (or `.cursor/plans/{plan-slug}-Code_Review_YYYY-MM-DD_HHMM.md`).

1. **Paste full report in this conversation.**
2. End with:
   - If clean: *"Code review clean. Run `/open-mr` to push, open the PR/MR, and monitor to merge."*
   - If findings need fixes: fix via worker subagent (`execute-plan`), then re-verify.
   - If plan-level gaps: *"Run `/update-plan` or `/converge-plan` before re-executing."*

## Pipeline position

| # | Skill | Actor |
| :--- | :--- | :--- |
| 1 | `/architect-bootstrap` | main agent (+ scan panel) |
| 2 | `/create-plan` | main agent (+ explorers) |
| 3 | `/pre-flight` | main agent (+ auditors, iterative) |
| 4 | `/update-plan` | main agent (+ conditional explorers) |
| 5 | `/converge-plan` | main agent (orchestrates pre-flight ⇄ update-plan) |
| 6 | `/execute-plan` | main agent (+ test specifier + executor) |
| 7 | `/codereview` | **main agent** (+ specialized review panel) |
| 8 | `/open-mr` | main agent (+ CI & review monitor) |
| 9 | `/eop` | main agent (terminal) |
