---
name: grill-me
description: >-
  Stage 3 of the agent pipeline (and conditional decision-resolution loop post /pre-flight
  or /converge-plan). Interviews the user relentlessly down each branch of a plan or design's
  decision tree — probing the codebase first, asking strictly one question per turn with a
  recommended stance — until shared understanding is reached and decisions are locked into
  the plan.
---

# Grill Me (Decision-Tree Alignment)

**Invoke via:** `/grill-me`

**You are the interviewer and systems architect.** Execute Steps 1–5 interactively with the user in the main session. **Never** dispatch `grill-me` as a background subagent.

**Mode:** **Plan** or **Agent**

**Pipeline entry points:**
1. **Post-`/create-plan` (Stage 3 — default design stress-test):** Walk down every unresolved branch of `{slug}.plan.md` before spending auditor passes on `/pre-flight` or `/converge-plan`.
2. **Post-`/pre-flight` or `/converge-plan` (Conditional blocker resolution):** Resolve `user-decision` items, stalled trade-offs, or ambiguous requirements surfaced by `/pre-flight`, `/update-plan`, or `/converge-plan`.
3. **Standalone:** Stress-test any feature brief, RFC, or architecture proposal.

**Pipeline:** `/create-plan` → **this** → `/pre-flight` (or `/converge-plan`) — and returned to conditionally from `/pre-flight` / `/converge-plan` whenever human decisions block readiness.

## Non-Negotiable Grilling Rules

1. **Explore the Codebase First:** If a question can be answered by reading source files, types, configs, tests, or `@ai-shebang` headers, probe the codebase yourself instead of asking the user.
2. **Depth-First Decision Tree (Dependency-Ordered):** Resolve upstream root decisions before downstream dependent branches; prune branches rendered moot by earlier answers.
3. **One Question at a Time:** Ask **strictly one** focused question per turn (using structured multiple-choice options when supported, or concise numbered options in chat). Never dump multi-question walls.
4. **Always Provide a Recommended Answer:** Put `(Recommended)` first on every question with a 1–2 sentence architectural rationale grounded in codebase evidence, Hexagonal boundaries, KISS, or Cynefin fit.
5. **Challenge Hidden Assumptions:** Probe distributed-system realities (idempotency, retry/backpressure, partial failure, migration safety, breaking contracts) and flag over-engineering.
6. **Stop When Converged:** End the interview once all material decision branches are resolved (or when the user says *"lock it in"* / *"good enough"*).

## Execution checklist

| Step | Who | Action | Gate |
| :--- | :--- | :--- | :--- |
| 1 | Main agent | **Context recall** (`AGENTS.md`, `.plan.md`, latest pre-flight / converge outcome) | Prior context loaded |
| 2 | Main agent | **Probe & map decision tree** (codebase first; list open branches) | Branch tree mapped |
| 3 | Main agent + User | **1-question-per-turn interview** (`(Recommended)` first, depth-first) | All branches resolved or user stops |
| 4 | Main agent | **Bake Locked Decisions** into `{slug}.plan.md` + `Revision History` | Plan updated in place |
| 5 | Main agent | Present Locked Decisions Ledger + CTA | Ready for `/pre-flight` or `/converge-plan` |

---

## Step 1: Context Recall

Read `AGENTS.md` (workspace and user-level if present), the target `.plan.md`, and — if invoked after an audit or convergence run — the latest `*-Pre-Flight-Review_*.md` and/or `*-converge-outcome_*.md` in `.agents/plans/` (or `.cursor/plans/`).

Identify which entry mode applies:
- **Post-`/create-plan`:** Focus on implicit design choices, boundary assumptions, failure modes, and 4-layer ATDD edge scenarios.
- **Post-`/pre-flight` / `/converge-plan`:** Seed the top of the decision tree with open `user-decision` / `blocked-on-user` rows and recurring stall blockers.

---

## Step 2: Codebase Probe & Decision-Tree Mapping

Before asking the user a single question:
1. Silently check every open question against the codebase (`read_file`, search, git history, tests, `@ai-shebang`).
2. Eliminate any branch already answered by existing code or `AGENTS.md`.
3. Order remaining open branches **depth-first by dependency** across:
   - **Scope & Non-Goals** (what is explicitly in/out)
   - **Contracts & Boundaries** (ports, adapters, schemas, backward compatibility)
   - **State, Failure & Rollback** (idempotency, retries, partial failure, migration safety)
   - **Verification & Edge Scenarios** (4-layer ATDD behaviors & fitness baselines)
   - **Audit Blockers** (any `user-decision` / stall items from `/pre-flight` or `/converge-plan`)

Announce the open branch count in one line and immediately ask **Question 1**.

---

## Step 3: Depth-First 1-Question-Per-Turn Interview

For each unresolved branch (one per turn):

1. State the branch and why it matters (1 sentence, referencing the relevant step or file).
2. Present 2–4 concrete options, placing **`(Recommended)`** first with a brief evidence-backed rationale.
3. Wait for the user's answer before asking the next question.
4. Prune or refine downstream branches immediately based on each answer.
5. Do **not** skip ahead to implementation or write production code.

---

## Step 4: Bake Locked Decisions into `{slug}.plan.md`

Once all material branches converge:

1. Build a compact **Locked Decisions Ledger**:

```markdown
| ID | Branch / Question | Locked Decision | Target Steps |
| :--- | :--- | :--- | :--- |
| D-1 | … | … | Step 2, Test T-3 |
```

2. Edit `{slug}.plan.md` **in place** (same path — never create a duplicate plan):
   - Replace tentative or ambiguous prose with the locked choices.
   - Sync YAML frontmatter `todos` if step scope or ordering changed.
   - Add or tighten 4-layer ATDD scenarios in Step 7 for newly clarified error/edge behaviors.
   - Add probe templates (`rules/plan-probes.md`) if any step shifted to Complex.
   - Append a row to `## Revision History` summarizing the locked decisions.
   - Enforce Path & Link Hygiene: zero `file:///` URLs and zero `/home/...` workstation paths.

---

## Step 5: Hand-Off CTA

Present the **Locked Decisions Ledger** in chat and end with:
- Post-`/create-plan`: *"Decisions locked into `{slug}.plan.md`. Run `/pre-flight` or `/converge-plan`."*
- Post-`/pre-flight` / `/converge-plan`: *"Blocker decisions locked into `{slug}.plan.md`. Re-run `/pre-flight` or `/converge-plan` to verify ≥ 90% readiness."*

---

## Invocation styles

```
/grill-me
```

Uses the newest `.plan.md` (and latest pre-flight/converge report if present).

```
/grill-me .agents/plans/{slug}.plan.md
```

Pinned plan path.

```
/grill-me

Focus on the retry/idempotency and migration rollback branches.
```

Targeted grill on specific decision branches.

---

## Anti-Patterns

- Asking multiple unrelated questions in a single turn
- Asking the user facts that reading the codebase would answer
- Asking open-ended questions without a `(Recommended)` option and rationale
- Leaving resolved decisions only in chat without updating `{slug}.plan.md`
- Dispatching `/grill-me` as a background subagent
- Introducing `file:///` links or `/home/...` workstation paths into the plan

## Pipeline position

| # | Skill | Actor |
| :--- | :--- | :--- |
| 1 | `/architect-bootstrap` | main agent (+ scan panel) |
| 2 | `/create-plan` | main agent (+ explorers) |
| 3 | `/grill-me` | **main agent** (interactive interview — also re-entered on audit/converge blockers) |
| 4 | `/pre-flight` | main agent (+ auditors, iterative) |
| 5 | `/update-plan` | main agent (+ conditional explorers) |
| 6 | `/converge-plan` | main agent (orchestrates pre-flight ⇄ update-plan) |
| 7 | `/execute-plan` | main agent (+ test specifier + executor) |
| 8 | `/codereview` | main agent (+ review panel) |
| 9 | `/open-mr` | main agent (+ CI & review monitor) |
| 10 | `/eop` | main agent (terminal) |
