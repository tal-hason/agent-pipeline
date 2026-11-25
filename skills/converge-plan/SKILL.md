---
name: converge-plan
description: >-
  Stage 5 of the agent pipeline. Automates iterative /pre-flight and /update-plan convergence
  rounds toward the ≥90% readiness gate with stall detection and a strict human execution gate.
---

# Converge Plan

**Invoke via:** `/converge-plan`

**You are the coordinator.** Run the Path-to-Green loop in this session until a stop condition fires. Main agent owns the deliverable — see [Agent Pipeline Contract](../agent-pipeline/SKILL.md).

**Purpose:** Automate `/pre-flight` → `/update-plan` → `/pre-flight` … toward the ≥90% soft gate without requiring the user to re-invoke each stage.

**Requires:** A `.plan.md` from `/create-plan` (path or @-mention). Optional: max rounds, user decisions, plan path in the invoke message.

**Mode:** **Agent** (dispatches auditors + conditional explorers).

**Hard rule:** **Never** run `/execute-plan`. Convergence ends at Ready (or a pause/abort). The human decides when to execute.

**Pipeline:** `/create-plan` → **this** (or `/pre-flight` alone) → `/execute-plan`

## Defaults

| Knob | Default | Override |
| :--- | :--- | :--- |
| Max rounds | **3** | `/converge-plan max=4` or "max 4 rounds" |
| Ready threshold | ≥ **90%** and status not Stop | soft gate from [Agent Pipeline Contract](../agent-pipeline/SKILL.md) |
| Stall | confidence **flat or down** vs previous round **and** ≥1 Path-to-Green blocker still present in plan text | — |
| Auto-execute | **off** (always) | not overridable |

One **round** = one full `/pre-flight` procedure. `/update-plan` runs between rounds when not Ready.

```
Round 1:  /pre-flight
          └─ if not Ready → /update-plan
Round 2:  /pre-flight
          └─ if not Ready → /update-plan
Round 3:  /pre-flight
          └─ stop (Ready / max / stall / pause / abort)
```

## Execution checklist

| Step | Who | Action | Gate |
| :--- | :--- | :--- | :--- |
| 1 | Main agent | Read `AGENTS.md`, resolve plan slug, max rounds, prior pre-flight trend | Loop config set |
| 2 | Main agent | **Loop:** run Round N per [Round procedure](#step-2-round-procedure) | Stop condition or next round |
| 3 | Main agent | Write converge outcome + present dashboard | User sees next CTA |

---

## Step 1: Context & Loop Configuration

1. Read `AGENTS.md` (workspace and user-level if present) for known pitfalls and constraints.
2. Resolve `{slug}.plan.md` (invoke path, @-mention, or newest relevant plan in `.agents/plans/` or `.cursor/plans/`).
3. List existing `*-Pre-Flight-Review_*.md` for the slug (confidence trend).
4. Set `max_rounds` (default **3**).
5. Note any user decisions from the invoke message — pass them into every `/update-plan` as User Input.
6. Present a one-line start banner: *"Converging `{slug}` — max {N} rounds, gate ≥90%, no auto-execute."*

**Do not** skip Round 1 pre-flight just because an old report exists — the loop always starts with a fresh audit of the **current** plan text (unless the user explicitly says "continue from latest pre-flight" / "start at update-plan").

---

## Step 2: Round Procedure

For each round `N` from 1 to `max_rounds`:

### 2a. Pre-flight (mandatory each round)

Execute the **full** procedure in [pre-flight](../pre-flight/SKILL.md) (Steps 1–6): Context Package, 3 auditors, pessimistic merge, write timestamped report.

Announce: *"Converge round {N}/{max}: pre-flight…"*

Record into the **Converge Ledger**:

| Field | Value |
| :--- | :--- |
| Round | N |
| Report path | `…-Pre-Flight-Review_….md` |
| Confidence | N% |
| Status | Ready / Caution / Stop |
| Top blockers | short list |

### 2b. Stop checks (after every pre-flight)

Evaluate **in order** — first match wins:

| # | Condition | Action |
| :--- | :--- | :--- |
| 1 | Confidence ≥ 90% **and** status is not Stop | **Ready** — exit loop (success) |
| 2 | `/update-plan` would need a redesign / unknown blast radius | **Abort** — recommend `/create-plan`; exit |
| 3 | Path to Green / gaps require `user-decision` items with no answer in invoke message | **Pause** — list decisions; exit (do not guess) |
| 4 | `N > 1` and confidence **≤** previous round's confidence **and** ≥1 prior blocker still visible in plan text | **Stall** — exit; show recurring blockers |
| 5 | `N == max_rounds` | **Max rounds** — exit with latest Path to Green |
| 6 | Else | Continue to 2c |

**Stall nuance:** If confidence is flat but Path to Green is **empty** / only `defer-to-execute`, treat as Ready-enough → exit with CTA `/execute-plan` (human override path), noting residual deferrals.

### 2c. Update-plan (between rounds only)

If continuing:

1. Announce: *"Converge round {N}/{max}: update-plan…"*
2. Execute the **full** procedure in [update-plan](../update-plan/SKILL.md) (Amendment Ledger → apply in place → Coherence Gate).
3. Pass: latest pre-flight (full), prior converge ledger, any user decisions.
4. If update-plan itself pauses on `user-decision` → **Pause** the converge loop (same as stop check #3).
5. Proceed to round `N+1` (another pre-flight).

**Never** run update-plan after a successful Ready pre-flight inside this loop.

---

## Step 3: Converge Outcome

Write: `.agents/plans/{slug}-converge-outcome_YYYY-MM-DD_HHMM.md` (or `.cursor/plans/{slug}-converge-outcome_YYYY-MM-DD_HHMM.md`).

```markdown
# Converge Outcome: {slug}

| Field | Value |
| :--- | :--- |
| Rounds run | {K} / max {max} |
| Terminal reason | Ready \| Stall \| Max rounds \| Pause (user-decision) \| Abort (redesign) |
| Final confidence | {N}% ({Status}) |
| Final pre-flight | {path} |
| Plan path | {path} |

## Round ledger
| Round | Confidence | Status | Delta | Action after |
| :--- | :--- | :--- | :--- | :--- |
| 1 | … | … | — | update-plan / stop |
| 2 | … | … | +/− | … |

## Recurring blockers (if stall / max)
- …

## Open user decisions (if pause)
- …

## Next
{CTA}
```

Present the same dashboard **in this conversation** (compact table + CTA).

### Terminal CTAs

| Terminal reason | Say |
| :--- | :--- |
| Ready | *"Converged at {N}%. Run `/execute-plan` when ready."* |
| Stall | *"Convergence stalled ({rounds} rounds). Recurring blockers below — decide manually, then `/update-plan` or `/create-plan`, or override into `/execute-plan`."* |
| Max rounds | *"Hit max {max} rounds at {N}%. Review Path to Green; run `/update-plan` + `/pre-flight`, raise max, or `/execute-plan` with override."* |
| Pause | *"Paused for decisions: {list}. Re-run `/converge-plan` with answers, or `/update-plan` then `/converge-plan`."* |
| Abort | *"Blast radius needs redesign. Run `/create-plan`."* |

---

## Invocation styles

```
/converge-plan
```

Newest/relevant plan; max 3 rounds.

```
/converge-plan .agents/plans/{slug}.plan.md
```

Pinned plan.

```
/converge-plan max=4
```

Raise round budget.

```
/converge-plan

Decisions: use Postgres full-text search; drop Step 11; accept naive circuit-breaker.
```

Feeds `/update-plan` User Input every round.

```
/converge-plan

Continue from latest pre-flight — start at update-plan.
```

Skip Round-1 audit only when user explicitly requests; still count the following pre-flight as round 1 for max/stall math, or label rounds as `U1, P1, U2, P2…` in the ledger and keep max **pre-flight** count at 3.

---

## Anti-Patterns

- Auto-invoking `/execute-plan` after Ready
- Silent guessing on `user-decision` rows to keep the loop going
- Skipping `/update-plan` and re-running `/pre-flight` on unchanged plan text
- Treating Path to Green as done without baking into `.plan.md`
- Ignoring stall (same blockers, flat confidence) and burning max rounds
- Starting a second converge loop while one is in progress in the same chat
- Using time-based polling loops for this workflow
- Averaging auditor scores (nested pre-flight remains pessimistic-min)
- Deliverable only in chat with no converge-outcome file

## Pipeline position

| # | Skill | Actor |
| :--- | :--- | :--- |
| 1 | `/architect-bootstrap` | main agent (+ scan panel) |
| 2 | `/create-plan` | main agent (+ explorers) |
| 3 | `/pre-flight` | main agent (+ auditors) — also nested inside converge |
| 4 | `/update-plan` | main agent — also nested inside converge |
| 5 | `/converge-plan` | **main agent** (orchestrates 3⇄4 until stop) |
| 6 | `/execute-plan` | main agent (+ test specifier + executor) |
| 7 | `/codereview` | main agent (+ review panel) |
| 8 | `/open-mr` | main agent (+ CI & review monitor) |
| 9 | `/eop` | main agent (terminal) |
