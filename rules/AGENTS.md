# Agent Pipeline — AGENTS.md

Workspace-specific memory for the agent pipeline. This file is maintained by `/eop` (End of Pipeline) and cross-session learnings.

---

## Agent pipeline (stable)

**Canonical contract:** `skills/agent-pipeline/SKILL.md`

**Flow:** `/architect-bootstrap` → `/create-plan` → `/pre-flight` ⇄ `/update-plan` (or `/converge-plan`) → `/execute-plan` → `/codereview` → `/open-mr` → `/eop`

| Stage | Main agent owns | Workers |
| :--- | :--- | :--- |
| `/architect-bootstrap` | Canvas + situational brief | Scan panel A–C (+ conditional D–J); ideation in main agent (Step 2.5) |
| `/create-plan` | `.plan.md` | 3× explorers (`ce-repo-research-analyst`, `explore`, `ce-correctness-reviewer`) |
| `/pre-flight` | Pre-Flight Dashboard | 3× `pre-flight` (+ optional `ce-security-lens-reviewer`) |
| `/update-plan` | Updated `.plan.md` + Amendment Ledger | Conditional 0–2 explorers (`explore`, `ce-correctness-reviewer`) for `evidence-gap` only |
| `/converge-plan` | Converge outcome + round ledger | Orchestrates nested `/pre-flight` + `/update-plan` |
| `/execute-plan` | Status + contract lock + todo sync + reconciliation | Red-First ATDD: `execute-plan-tests` (Red harness) → `execute-plan` (Green SUT) |
| `/codereview` | Code Review report | R1–R6 core + conditional `ce-*` panel |
| `/open-mr` | PR/MR + monitor outcome | None (`gh` / `glab` CLI monitor) |
| `/eop` | Session digest + `AGENTS.md` delta | None (terminal) |

**Recommended mode per stage:**

| Stage | Mode |
| :--- | :--- |
| `/architect-bootstrap` | Agent or Ask (Ask for fresh ideas) |
| `/create-plan` | Plan |
| `/pre-flight` | Agent or Plan |
| `/update-plan` | Plan (Agent if explorers needed) |
| `/converge-plan` | Agent |
| `/execute-plan` | Agent |
| `/codereview` | Agent |
| `/open-mr` | Agent |
| `/eop` | Agent |

**Rules:** Read `AGENTS.md` and prior artifacts on disk first; pass Context Packages by Reference in worker prompts; telegraphic (caveman) worker output; delegate all code mutations to worker subagents (`rules/orchestrator-mandate.md`); thinking-tier models for audit/review workers.

**References (not subagents):** `references/architect-brief-canvas.md`

**Team how-to:** `docs/pipeline-howto.md` — onboarding guide for the pipeline workflows.

**`/pre-flight` is iterative** — new timestamped report each run; `/execute-plan` uses latest; soft gate ≥ 90%. Default path to green: `/update-plan` (bake Path to Green into the plan) → `/pre-flight` again, or **`/converge-plan`** to automate that loop (max 3 rounds, stall detection, never auto-execute).

---

## Learned preferences

_Add stable, cross-session preferences below. `/eop` appends here._

- Commit only when explicitly confirmed (e.g., "yes" after being prompted).
- When documenting pipeline commands or modes, describe behavior factually — don't editorialize or characterize choices without the user saying so.
- Preferred pipeline workflow uses two sessions: Session 1 (exploration) scans the workspace and converges on a solution, then outputs a crafted prompt; Session 2 (execution) starts fresh with distilled intent for `/architect-bootstrap` onward — disk artifacts and `AGENTS.md` bridge the gap.

### Auto-merge + Automated PR/MR Bot Commits

When a PR/MR has auto-merge ("merge when pipeline succeeds") enabled and its CI pipeline includes an automated bot with commit/auto-publish capability, a green pipeline does not guarantee the auto-committed content is correct — each bot pass can introduce a fresh regression that gets merged before a human reviews it. When monitoring this kind of PR/MR: diff every bot auto-commit yourself before trusting the next green pipeline, and disable auto-merge on follow-up fix PRs/MRs until the bot behavior is verified.
