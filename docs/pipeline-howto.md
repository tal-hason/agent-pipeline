# How to Use the Agent Pipeline

A team guide for the 9-stage agent pipeline that turns a vague idea into reviewed, shipped, and documented code.

## The Pipeline at a Glance

```
/architect-bootstrap → /create-plan → /pre-flight ⇄ /update-plan → /execute-plan → /codereview → /open-mr → /eop
     (WHERE+WHAT)         (HOW)            ▲             │               (build)        (review)     (ship)    (close)
                                           └─ /converge-plan ─┘
```

Each stage feeds into the next. You can enter the pipeline at any point, but earlier stages give the agent better context. `/update-plan` bakes Path to Green into the plan; `/converge-plan` runs `/pre-flight` ⇄ `/update-plan` until Ready (or stall / max rounds / pause) without auto-executing. `/open-mr` pushes the branch, creates the PR/MR, monitors CI and reviews, and merges when safe.

| Stage | Recommended Mode | What it does | Output |
|---|---|---|---|
| `/architect-bootstrap` | **Agent** or **Ask** | Scans the workspace, discovers tech stack, boundaries, and risks. Optionally runs brainstorming if scope is unclear. | Interactive canvas + situational brief |
| `/create-plan` | **Plan** | Builds an atomic, step-by-step implementation plan with 4-layer ATDD test specifications. | `{slug}.plan.md` |
| `/pre-flight` | **Agent** or **Plan** | Audits the plan for risks, gaps, and feasibility. Iterative — run as many times as needed. | Timestamped review with confidence % |
| `/update-plan` | **Plan** (Agent if explorers needed) | Bakes Path to Green into `.plan.md` (surgical amendments + targeted evidence when needed). | Updated plan + Amendment Ledger |
| `/converge-plan` | **Agent** | Automates `/pre-flight` ⇄ `/update-plan` until ≥90% Ready, stall, max rounds (default 3), or user-decision pause. Never auto-executes. | Converge outcome + nested reports |
| `/execute-plan` | **Agent** | Implements the plan via Red-First ATDD (failing test harness → contract lock → production code). Soft gate: needs pre-flight >= 90%. | Sealed tests, code changes, & execution outcome |
| `/codereview` | **Agent** | Multi-reviewer panel checks the diff against the plan, security invariants, and coding standards. | Merged review report |
| `/open-mr` | **Agent** | Pushes branch, opens PR/MR (`gh` or `glab`), monitors CI & review comments, merges when safe. | PR/MR URL + outcome |
| `/eop` | **Agent** | Updates project docs, optionally updates linked issues/PRs, extracts learnings into `AGENTS.md`, and writes a session digest. | Session digest + `AGENTS.md` delta + doc updates |

### Choosing the Right Mode

Not every stage needs Agent mode. The mode should match the nature of the work:

- **`/architect-bootstrap`** — Use **Ask mode** when exploring a fresh idea or unfamiliar codebase (read-only reconnaissance). Switch to **Agent mode** when you want the full scanning pipeline with canvas generation and subagents.
- **`/create-plan`** — Use **Plan mode**. Planning is a collaborative design activity, not an execution task. Plan mode keeps the focus on thinking through the approach before committing to implementation.
- **`/pre-flight`** — Either **Agent** or **Plan** works. Use Agent mode when you want the full auditor panel dispatched. Use Plan mode when you want to review and discuss the plan collaboratively before auditing.
- **`/update-plan`** — Prefer **Plan mode**. Rewrites the plan from Path to Green; switches to Agent only when targeted explorers must read the repo.
- **`/converge-plan`** — **Agent mode** only. Orchestrates nested `/pre-flight` + `/update-plan` rounds until a stop condition.
- **`/execute-plan`** — **Agent mode** only. Coordinates Red-First ATDD workers, contract locking, and verification.
- **`/codereview`** — **Agent mode** only. The review panel dispatches subagents and writes report files.
- **`/open-mr`** — **Agent mode** only. Pushes the branch, monitors CI/reviews, and merges when gates pass.
- **`/eop`** — **Agent mode** only. Updates docs, `AGENTS.md`, and writes the session digest.

---

## `/architect-bootstrap` — Invocation Styles

This is where most sessions start. The command adapts its behavior based on how much context you give it.

### Style 1: Just the command (scan only)

```
/architect-bootstrap
```

**When to use:** You want the agent to scan whatever workspace is currently open.

**What happens:**
- Scans the current workspace root(s)
- Discovers tech stack, folder structure, dependencies, CI/CD surfaces
- Produces a canvas with situational awareness
- Does NOT start brainstorming or ideation (no feature request given)

**Typical follow-up:** Review the canvas, then either:
- Run `/create-plan` if you already know what to build
- Ask the agent to brainstorm if you need direction

---

### Style 2: Command + folder/path (targeted scan)

```
/architect-bootstrap ~/Git/my-service
```

**When to use:** You want to bootstrap a specific repo or folder, not the entire workspace.

**What happens:**
- Focuses scanning on the specified path
- Reads that repo's `README`, `AGENTS.md`, manifests, and build surfaces
- Produces a canvas scoped to that project
- Still no feature intent — pure reconnaissance

**Typical follow-up:** Same as Style 1. You now have a situational brief for a specific repo and can plan work against it.

---

### Style 3: Command + folder/path + feature/fix request (full pipeline entry)

```
/architect-bootstrap ~/Git/my-service

Add a health check endpoint that returns pod readiness and database connectivity status
```

```
/architect-bootstrap ~/Git/my-service

Fix the retry logic in the event processor -- it silently drops events after 3 failures instead of sending to DLQ
```

```
/architect-bootstrap ~/Git/my-repo

I want to create a how-to doc for my team on using the pipeline workflows
```

**When to use:** You know BOTH where the work is AND what you want to do. This is the most efficient entry point.

**What happens:**
- Scans the specified path (WHERE lane)
- The feature/fix description becomes the User Intent (WHAT lane)
- If the intent is clear and bounded, scanning proceeds immediately
- If the intent is vague ("improve performance", "make it better"), the agent runs an interactive brainstorm before scanning
- Produces a canvas that combines workspace reality + scoped intent
- Scanners validate whether the codebase supports the requested change

**Typical follow-up:** Run `/create-plan` — the bootstrap brief carries directly into planning.

---

### Style 4: Command + vague idea (triggers brainstorming)

```
/architect-bootstrap ~/Git/my-platform

I want to improve the developer experience
```

**When to use:** You have a general direction but no concrete requirements.

**What happens:**
- The agent detects vague intent and enters the ideation lane (Step 2.5)
- Runs an interactive Q&A to sharpen the idea (one question per turn)
- Produces a requirements document in `docs/brainstorms/`
- Then proceeds with the workspace scan using those requirements as the anchor
- Scanners validate the brainstormed requirements against codebase reality

**Typical follow-up:** Review the requirements doc, then `/create-plan`.

---

## The Rest of the Pipeline

### `/create-plan`

```
/create-plan
```

Run after bootstrap. The agent reads the canvas brief and builds an atomic plan.

- Dispatches 3 evidence explorers to validate assumptions
- Produces a `.plan.md` with numbered steps, affected files, and a 4-layer ATDD Test Specification
- Presents the full plan for your review before proceeding

**Tip:** If bootstrap produced a brainstorm requirements doc, the plan will reference it.

### `/pre-flight`

```
/pre-flight
```

Run after the plan is written. Audits the plan for risks.

- Dispatches 3 auditors who independently score confidence
- Produces a timestamped review report with a confidence percentage
- **Iterative** — run it as many times as you want. Each run creates a new report.
- Soft gate: `/execute-plan` expects >= 90% confidence

**Tip:** If pre-flight finds gaps, run `/update-plan` or `/converge-plan` (not ad-hoc chat edits). No penalty for re-runs.

### `/update-plan`

```
/update-plan
```

Run when pre-flight (or code review) leaves Path to Green / plan gaps.

- Loads the full plan + latest pre-flight into an Update Context Package
- Builds an Amendment Ledger (`plan-text` / `scope` / `evidence-gap` / `user-decision` / `defer-to-execute`)
- Applies fixes **in place** into atomic steps (replaces stale text — does not stack)
- Spawns targeted explorers only for `evidence-gap` rows
- Syncs frontmatter todos + Test Spec; appends Revision History
- Ends with a diff summary and usually *"Run `/pre-flight` again"*

**Tip:** A common failure mode is documenting Path to Green in the pre-flight report without rewriting the plan. `/update-plan` closes that loop before the next audit. For multiple rounds without re-invoking, use `/converge-plan`.

```
/update-plan

Decisions: use Postgres full-text search; drop Step 11 from scope; accept the naive circuit-breaker trade-off.
```

Pass user decisions in the invoke message when the ledger would otherwise block on `user-decision` rows.

### `/converge-plan`

```
/converge-plan
```

Automates the path to the ≥90% soft gate after a plan exists.

- Runs `/pre-flight` → `/update-plan` → `/pre-flight` … in one session
- Stops on Ready (≥90%, not Stop), stall (flat/down confidence + recurring blockers), max rounds (default 3), user-decision pause, or redesign abort
- **Never** auto-runs `/execute-plan`
- Writes `{slug}-converge-outcome_*.md` with a round ledger

```
/converge-plan max=4

Decisions: use Postgres full-text search; drop Step 11.
```

**Tip:** Use `/converge-plan` when you want hands-off re-scoring. Use `/update-plan` alone when you want to inspect one amendment pass before the next audit.

### `/execute-plan`

```
/execute-plan
```

Run after pre-flight passes (>= 90%). Implements the plan via Red-First ATDD.

- Checks the latest pre-flight report (if below 90%, shows blockers and suggests `/converge-plan` or `/update-plan` → `/pre-flight`, or override)
- Phase 4A: `execute-plan-tests` writes 4-layer acceptance tests and proves they fail (**RED**)
- Phase 4B: Locks test files as an immutable contract
- Phase 4C: `execute-plan` writes production code until all locked tests pass (**GREEN**)
- Reconciles test integrity and syncs plan frontmatter todos

### `/codereview`

```
/codereview
```

Run after execution completes. Reviews the diff.

- Dispatches a panel of specialized reviewers (correctness, maintainability, security, testing, etc.)
- Merges findings into a unified report with severity levels
- If clean: run `/open-mr`
- If plan-level issues found: loop back to `/update-plan` or `/converge-plan`

### `/open-mr`

```
/open-mr
```

Run after code review is clean (or with override). Ships the branch.

- Pushes the branch and creates the PR/MR (`gh` or `glab` auto-detected)
- Optionally requests reviewers/labels
- Monitors CI checks and review comments, verifying any findings against source code
- Merges when green and safe (or pauses if human-gated / `no-merge`)
- Does **not** enable auto-merge on repos where automated bots push commits to PR/MR branches

```
/open-mr https://github.com/org/repo/pull/123
```

Monitor an existing PR/MR (skip create).

```
/open-mr

Do not merge — notify only when green.
```

### `/eop`

```
/eop
```

Run after `/open-mr` merges (or when closing early). Closes the pipeline.

- **Doc Updates**: reviews and updates project `README.md`, `CHANGELOG`, and architecture docs with verified session outcomes
- **Issue & PR Status**: optionally adds a completion comment or closes linked issues/PRs upon confirmation
- **Continual Learning**: mines the session for durable learnings (preferences, workspace facts, workflow patterns, anti-patterns) and appends them to `AGENTS.md`
- **Context Compression**: writes a session digest summarizing pipeline stages completed, key decisions, artifacts produced, and open threads
- This is the terminal stage — no further commands follow

**Tip:** Even if you didn't run the full pipeline, `/eop` works after any partial run. It captures whatever learnings the session produced.

---

## Quick Reference

| I want to... | Start with |
|---|---|
| Explore a new repo I just cloned | `/architect-bootstrap ~/Git/repo` |
| Fix a specific bug | `/architect-bootstrap ~/Git/repo` + describe the bug |
| Build a new feature | `/architect-bootstrap ~/Git/repo` + describe the feature |
| Figure out what to build | `/architect-bootstrap ~/Git/repo` + vague idea (triggers brainstorm) |
| Plan without scanning | `/create-plan` (skip bootstrap) |
| Re-audit after plan changes | `/pre-flight` (iterative, run again) |
| Apply Path to Green into the plan | `/update-plan` then `/pre-flight` |
| Auto-loop to ≥90% soft gate | `/converge-plan` |
| Review code after manual changes | `/codereview` (works on any staged diff) |
| Open PR/MR, watch CI & reviews, merge | `/open-mr` |
| Close a session and preserve context | `/eop` (works after any stage) |

## Creator's Recommended Workflow

The most effective way to use the pipeline is a **two-session approach**: an exploration session to build understanding, then a focused execution session to run the pipeline.

### Session 1: Explore and Formulate (the "main chat")

**Mode: Agent** (needed for bootstrap scanning and canvas generation)

Start a vague, open-ended chat pointing at your workspace or folder. The goal is NOT to build anything yet — it's to learn.

```
/architect-bootstrap ~/Git/my-service
```

Use this session to:

1. **Explore the codebase.** Ask questions, read through scan results, understand boundaries and dependencies.
2. **Build situational awareness.** Let the agent surface the tech stack, architecture, gaps, and risks. Ask follow-up questions until you have a mental model of the current state.
3. **Formulate intent.** Once you understand where things stand, describe what you want to achieve — even loosely. Brainstorm with the agent. Refine until the solution direction is clear.
4. **Request a feature prompt.** Ask the agent to write a crisp, self-contained prompt for `/architect-bootstrap` that captures the solution you've converged on. Something like:

   > "Write me an architect-bootstrap prompt for this feature that I can use in a fresh session."

   The agent will produce a prompt that bundles context, intent, scope, and constraints — everything the next session needs to hit the ground running.

### Session 2: Execute the Pipeline (fresh context)

**Mode: Agent**

Open a **new chat session**. Paste the feature prompt from Session 1:

```
/architect-bootstrap ~/Git/my-service

<paste the feature prompt from Session 1>
```

This gives the agent a clean context window with a sharp, well-defined intent. From here, follow the pipeline:

1. **Bootstrap** scans the workspace against your specific feature prompt
2. **Brainstorm / focus / probe** — if the agent enters ideation, work through it. Sharpen scope, challenge assumptions, validate feasibility against the codebase
3. **`/create-plan`** — only run this after you're confident the intent is solid and the bootstrap canvas reflects reality
4. **`/pre-flight`** — audit the plan (single pass)
5. **`/converge-plan`** — optional: auto-loop `/pre-flight` ⇄ `/update-plan` to ≥90% (or run them manually)
6. **`/execute-plan`** — implement via Red-First ATDD
7. **`/codereview`** — review
8. **`/open-mr`** — open PR/MR, monitor CI & reviews, merge when safe
9. **`/eop`** — capture learnings into `AGENTS.md` and compress context for next time

### Why Two Sessions?

- **Session 1 gets noisy.** Exploration generates a long context with false starts, tangents, and evolving understanding. That's valuable for learning but counterproductive for execution.
- **Session 2 starts clean.** The feature prompt distills everything Session 1 discovered into a focused starting point. The agent doesn't inherit noise — it inherits intent.
- **Artifacts bridge the gap.** The bootstrap canvas, `docs/brainstorms/` requirements, `.agents/plans/` reports, and `AGENTS.md` persist on disk across sessions.

---

## Tips

1. **You can enter at any stage.** Bootstrap is recommended but not required. `/create-plan` can work without a canvas.
2. **Pre-flight is iterative.** Run it multiple times. Each run produces a new timestamped report. The agent uses the latest one.
3. **Update the plan between audits.** Prefer `/update-plan` so Path to Green lands in the steps the executor will read. Use `/converge-plan` to automate multiple rounds.
4. **Override gates when needed.** The 90% pre-flight gate is soft — you can tell the agent to proceed anyway.
5. **Stages chain in one session.** Run the whole pipeline in a single chat, or spread across sessions (disk artifacts and `AGENTS.md` carry context).
6. **Artifacts persist on disk.** Each stage writes its output (`.plan.md`, pre-flight reports, execution outcomes, code reviews, digests) so subsequent stages and future sessions can read them directly.
