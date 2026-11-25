---
name: pre-flight
description: >-
  Stage 4 of the agent pipeline. Audits an implementation plan (.plan.md)
  for confidence, boundary violations, integration risks, and Cynefin alignment before touching
  production code. Soft gate ≥ 90%. Use iteratively to validate and refine plans prior to execution.
---

# Pre-Flight Review

**Invoke via:** `/pre-flight`

**You are the coordinator.** Execute Steps 1–6 in order yourself. Main agent owns the deliverable — see [Agent Pipeline Contract](../agent-pipeline/SKILL.md).

Spawn **`pre-flight` auditors** in Step 3 (`readonly: true`). You merge their output and author the unified report in **this** conversation.

**Requires:** A `.plan.md` from `/create-plan` (path or @-mention).

**Iterative:** Run this skill **as many times as needed** — each run writes a new timestamped report. Re-runs are encouraged, not penalized.

## Execution checklist

| Step | Who | Action | Gate |
| :--- | :--- | :--- | :--- |
| 1 | Main agent | **Context recall** (`AGENTS.md`, plan, prior pre-flight reports) | Prior trend loaded |
| 2 | Main agent | Build **Pre-Flight Context Package** | Full plan loaded — not an excerpt |
| 3 | Auditors (parallel) | Dispatch `pre-flight` subagents (+ optional security lens) | All auditors returned |
| 4 | Main agent | Pessimistic merge + correlation | Report draft ready |
| 5 | Main agent | Write `*-Pre-Flight-Review_*.md` | File on disk |
| 6 | Main agent | Present dashboard | User sees confidence + path to green |

**Soft gate for `/execute-plan`:** ≥ 90% and not Stop → ready; if Path to Green has `user-decision` / ambiguous trade-offs → **`/grill-me`**; otherwise **`/update-plan`** or **`/converge-plan`**, then re-audit, or user override (see [Agent Pipeline Contract](../agent-pipeline/SKILL.md)).

**Next:** Ready → `/execute-plan` → `/codereview`. User decisions needed → `/grill-me`. Else → `/update-plan` / `/converge-plan`.

---

## Step 1: Context Recall

Read `AGENTS.md` (workspace and user-level if present), the target `.plan.md`, and all prior `*-Pre-Flight-Review_*.md` reports for this plan slug. Note any Revision History / Amendment Ledger deltas if `/update-plan` ran since the last audit.

---

## Step 2: Pre-Flight Context Package (by Reference)

**Main agent only.** Read the plan file from disk. Build this package — auditors receive the **package by reference** with exact file paths, avoiding prompt token bloat from duplicating large plan files:

```markdown
## Pre-Flight Context Package: {plan-slug}

### 1. Plan Identity
- **Path:** absolute path to `.plan.md` (read directly via file tools)
- **Name / overview:** from YAML frontmatter
- **Workspace:** current workspace root

### 2. Plan Structure Summary
- Numbered atomic steps with Cynefin tags from the plan
- Complex / probe-required steps highlighted
- Files referenced in the plan (repo-relative paths)

### 3. Prior Art & Iteration History
- Relevant `AGENTS.md` constraints from Step 1
- Prior `*-Pre-Flight-Review_*.md` for this plan slug (summarize confidence trend)
- If plan changed via `/update-plan`: note Revision History / Amendment Ledger summary

### 4. Audit Mission
Stress-test the plan above. Classify each step (Cynefin), score confidence, find gaps.
Audit path & link hygiene: verify zero `file:///` URLs and zero leaked `/home/<user>/...`
developer paths; internal targets must be repo-relative and external links canonical HTTPS Git URLs.
Do not execute code. Read the plan from its path directly.
```

**Gate:** Package complete before Step 3. Never dispatch auditors with only the user's last message.

---

## Step 3: Parallel Audits

**WAIT** for all auditors before Step 4.

Dispatch **3** `pre-flight` subagents in **one message** using distinct thinking-tier models where supported. `readonly: true`.

| Auditor | Model | Lens |
| :--- | :--- | :--- |
| **A** | Thinking-tier (distinct model) | Cynefin + architecture depth, integration patterns |
| **B** | Thinking-tier (distinct model) | Gap analysis, missing context, plan completeness, path & link hygiene |
| **C** | Thinking-tier (distinct model) | Contracts, platform abstractions, edge cases |

**Optional security lane (4th auditor):** When the plan touches auth/authz, secrets, public endpoints, PII, or trust boundaries — dispatch **`ce-security-lens-reviewer`** in the **same message** (`readonly: true`). Mission: plan-level threat model gaps (not code audit). Findings merge into Gap Analysis and Path to Green.

**Model policy:** Thinking-tier only. No `*-fast` models.

### Each prompt MUST include

1. **Pre-Flight Context Package by Reference** (pass plan path)
2. **Mission:** execute the `pre-flight` agent procedure; return structured Pre-Flight Dashboard only
3. **Communication Contract:** strict telegraphic style (caveman), zero greetings/filler/intros, start directly with `### 1. Confidence Dashboard`
4. **Output contract:** per `agents/pre-flight.md` output format

---

## Step 4: Merge (Pessimistic)

| Rule | Policy |
| :--- | :--- |
| Confidence per task | **Minimum** across auditors |
| Overall confidence | **Minimum** across auditors |
| Cynefin | **More complex** wins |
| Gaps / blockers | **Union** — one auditor flag = real gap |
| Contradictions | Flag for user; do not silently resolve |
| Path to Green | Merge, dedupe, blockers first |

**Cross-issue correlation:** Collapse findings that share one root cause before finalizing. Produce **Auditor agreement matrix** (task × A/B/C × confidence).

---

## Step 5: Write Report

Path: `.agents/plans/{plan-slug}-Pre-Flight-Review_YYYY-MM-DD_HHMM.md` (or `.cursor/plans/{plan-slug}-Pre-Flight-Review_YYYY-MM-DD_HHMM.md`).

Required sections:
1. Confidence Dashboard (overall %, Ready/Caution/Stop, auditors, top blockers)
2. Task-by-Task Analysis (min confidence, agreement column)
3. Integration Risk Matrix
4. Platform Abstraction Audit (if applicable)
5. Gap Analysis (< 95% tasks — ambiguity, missing context, risk, source auditor)
6. Security Threat Model (if security lane ran — auth assumptions, data exposure, missing controls)
7. Auditor Disagreements (if any)
8. Path to Green (blockers + suggestions)

---

## Step 6: Present Dashboard

1. **Paste the full Pre-Flight Dashboard in this conversation.**
2. State outcome (no pushback on re-runs):
   - *"Confidence {N}%. Run #{K} for this plan. {Ready → `/execute-plan` | User-decision gaps → `/grill-me` | Caution/Stop → Path to Green below; run `/update-plan` or `/converge-plan`}."*

---

## Iteration loop

```
/create-plan → /grill-me → /pre-flight → /update-plan → /pre-flight → … → /execute-plan
                                ▲    │            │
                                │    ▼            │
                                └─ /grill-me ─────┘  (if user-decision trade-offs block readiness)

Or: /create-plan → /grill-me → /converge-plan → /execute-plan
```

- Each `/pre-flight` run is independent — never say "you already ran pre-flight."
- Compare against prior reports; highlight **delta** in confidence and resolved blockers.
- Use **`/grill-me`** when Path to Green surfaces unresolved design trade-offs or `user-decision` items.
- Prefer **`/update-plan`** (single amend) or **`/converge-plan`** (auto loop) over ad-hoc plan edits.
- **Stop** status is advisory — user may `/grill-me` / `/update-plan` / `/converge-plan` immediately.

---

## Anti-Patterns

- Overlooking leaked developer home paths (`/home/...`) or `file:///` URLs in the plan during audit
- Pushback when user runs `/pre-flight` again
- Auditing from plan excerpt while full file exists on disk
- Averaging confidence scores
- Discarding a gap because only one auditor found it
- Deliverable only in subagent thread
- Telling the user to "edit the plan" without recommending `/update-plan` or `/converge-plan`

## Pipeline position

| # | Skill | Actor |
| :--- | :--- | :--- |
| 1 | `/architect-bootstrap` | main agent (+ scan panel) |
| 2 | `/create-plan` | main agent (+ evidence explorers) |
| 3 | `/grill-me` | main agent (interactive decision-tree interview) |
| 4 | `/pre-flight` | **main agent** (+ `pre-flight` auditors, iterative) |
| 5 | `/update-plan` | main agent (+ conditional explorers) |
| 6 | `/converge-plan` | main agent (orchestrates pre-flight ⇄ update-plan) |
| 7 | `/execute-plan` | main agent (+ test specifier + executor) |
| 8 | `/codereview` | main agent (+ `codereview` reviewers) |
| 9 | `/open-mr` | main agent (+ CI & review monitor) |
| 10 | `/eop` | main agent (terminal) |
