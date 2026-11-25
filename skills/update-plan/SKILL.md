---
name: update-plan
description: >-
  Stage 5 of the agent pipeline. Surgical plan updater that builds an Amendment Ledger
  from /pre-flight Path to Green or user decisions, dispatches conditional targeted explorers
  for evidence gaps, rewrites {slug}.plan.md in place, and enforces a Coherence Gate.
---

# Update Plan

**Invoke via:** `/update-plan`

**You are the coordinator.** Execute Steps 1–8 in order yourself. Main agent owns the deliverable — see [Agent Pipeline Contract](../agent-pipeline/SKILL.md).

**Purpose:** Bake Path to Green (and related gaps) into the live `{slug}.plan.md` so the next `/pre-flight` or `/execute-plan` reads a coherent plan — not a stale plan plus a separate amendment list.

**Requires:** A `.plan.md` from `/create-plan` (path or @-mention). Ideally a latest `*-Pre-Flight-Review_*.md`. Optional: user notes / decisions in the invoke message.

**Mode:** Prefer **Plan** mode (collaborative rewrite). Agent mode is fine when explorers must read the repo.

**Depth: Adaptive** — surgical by default; spawn targeted explorers only for `evidence-gap` items.

**Pipeline:** `/pre-flight` → **this** → `/pre-flight` (again) → `/execute-plan`

## Execution checklist

| Step | Who | Action | Gate |
| :--- | :--- | :--- | :--- |
| 1 | Main agent | **Context recall** (`AGENTS.md` + prior pre-flight reports) | Prior patterns & reports loaded |
| 2 | Main agent | Build **Update Context Package** | Full plan + latest pre-flight loaded |
| 3 | Main agent | Build **Amendment Ledger** | Every item classified; user-decisions surfaced |
| 4 | Explorers (conditional) | Targeted evidence only for `evidence-gap` rows | Evidence returned or deferred |
| 5 | Main agent | Apply amendments **in place** | Stale text replaced, not stacked |
| 6 | Main agent | Coherence gate | Checklist green (or explicit exceptions) |
| 7 | Main agent | Write plan + Revision History row | File on disk |
| 8 | Main agent | Present diff summary | User can re-run `/pre-flight` |

**Next:** `/pre-flight` or `/converge-plan` (default) — or `/execute-plan` if latest pre-flight was already ≥ 90% and this update only recorded deferrals / no material step changes.

---

## Step 1: Context Recall

Read `AGENTS.md` (workspace and user-level if present), the target `.plan.md`, and all prior `*-Pre-Flight-Review_*.md` reports for this plan slug from `.agents/plans/` (or `.cursor/plans/`).

Carry recurring pitfalls and prior amendment patterns into the Context Package.

---

## Step 2: Update Context Package

**Main agent only.** Read from disk — never from chat excerpts alone.

```markdown
## Update Context Package: {plan-slug}

### 1. Plan Identity
- Path, name/overview, workspace

### 2. Full Plan Text
(complete `.plan.md` — YAML frontmatter + entire body)

### 3. Latest Pre-Flight
- Path + timestamp of newest `*-Pre-Flight-Review_*.md`
- Overall confidence, Ready/Caution/Stop
- Full Path to Green + Gap Analysis + Security findings (if any)
- Task-by-task min confidence (for targeting)

### 4. Prior Pre-Flight Trend
- List prior reports for this slug: run #, confidence, status
- Note blockers that reappear across runs (signal: previous updates were incomplete)

### 5. User Input
- Explicit decisions, scope cuts, accepted trade-offs, or extra amendments from the invoke message
- If none: state "none — drive from Path to Green only"

### 6. Mission
Produce a coherent updated plan. Replace stale step text. Sync todos + Test Spec. Do not execute production code.
```

**If no pre-flight exists:** Ask whether to proceed from user notes only, or stop and recommend `/pre-flight` first. Do not invent Path to Green.

**Gate:** Sections 1–2 complete before Step 3. Section 3 required unless user explicitly overrides.

---

## Step 3: Amendment Ledger

Extract every actionable item from:

1. Latest Path to Green (all checkboxes / numbered fixes)
2. Gap Analysis rows that imply plan-text changes
3. Security Threat Model blockers (if present)
4. User input (Step 2 §5)
5. Recurring blockers from prior runs that are still visible in the current plan text

### Classification (exactly one per item)

| Class | Meaning | Default action |
| :--- | :--- | :--- |
| `plan-text` | Precision fix in existing step/prose/pseudocode/verify command | Edit in place |
| `scope` | Add/split/merge/drop steps, files, or Test Spec rows | Restructure plan + frontmatter |
| `evidence-gap` | Claim needs codebase confirmation before rewriting | Step 4 explorer(s), then edit |
| `user-decision` | Ambiguous choice (provider, trade-off, keep/drop scope) | **Stop and ask** — do not guess |
| `defer-to-execute` | Implementation-time detail; plan already executable | Record in Revision History only; **do not** dilute steps |

### Ledger format (required)

```markdown
| ID | Source | Class | Amendment | Target (step / section) | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| A-1 | PtG #1 | plan-text | … | Step 3 pseudocode | pending |
| A-2 | Gap G-3 | evidence-gap | … | Step 6 URL chain | needs-evidence |
| A-3 | User | user-decision | Postgres full-text vs external search | Step 8 | blocked-on-user |
```

**Gate:**

- Any `user-decision` → present options, **wait for user** before applying those rows (or recommend `/grill-me` when multiple interdependent design decisions remain).
- Do not mark `defer-to-execute` for items that change contracts, file lists, or control flow — those are `plan-text` or `scope`.

### Anti-pattern this step exists to prevent

Documenting fixes only in chat or only in a new pre-flight appendix while leaving **broken/stale plan text** unchanged. The next `/pre-flight` will re-find the same bugs and confidence will stall.

---

## Step 4: Targeted Evidence (conditional)

Spawn explorers **only** for open `evidence-gap` rows. Skip this step entirely if none.

| When | Workers | Rules |
| :--- | :--- | :--- |
| 1–2 gaps, narrow files | 1× `explore` (`thoroughness: medium`) | Readonly; return evidence only |
| ≥3 gaps or deep call chains | Up to 2: `explore` + `ce-correctness-reviewer` | Parallel; thinking-tier models; `readonly: true` |
| Full redesign / unknown blast radius | Stop | Recommend `/create-plan` instead of `/update-plan` |

Each explorer prompt MUST include:

1. The specific ledger rows + suspected files
2. Relevant excerpts from the Update Context Package (not a one-liner)
3. Mission: *"Return evidence to resolve these gaps. Do not rewrite the plan."*

After evidence returns: update ledger Status (`ready` / `still-blocked`). Still-blocked → ask user or reclassify as `user-decision`.

---

## Step 5: Apply Amendments In Place

Edit `.agents/plans/{slug}.plan.md` (or `.cursor/plans/{slug}.plan.md` — same file; do not create a parallel plan).

### Rewrite rules

1. **Replace, don't stack** — remove stale pseudocode, wrong signatures, obsolete claims; leave one authoritative version.
2. **Bake into steps** — Path to Green items become part of Atomic Execution Steps / strategy / verification — not a sticky note at the bottom alone.
3. **Sync dependents** when steps change:
   - YAML frontmatter `todos` (one todo per atomic step; Cynefin tag in content)
   - 4-layer Test Specification table (add/adjust rows for new behaviors)
   - Mermaid diagram (only if topology/scope changed)
   - Explorer agreement / evidence summary (only if new evidence landed)
4. **Complex steps** — if a step becomes Complex, add probe template (`rules/plan-probes.md`).
5. **Preserve completed work** — do not reset `status: completed` todos unless the step content materially changed (then set `pending` and note why in Revision History).
6. **Revision History** — append a section (create if missing) with one row for this update:

```markdown
## Revision History

| Update | Source pre-flight | Confidence then | Summary |
| :--- | :--- | :--- | :--- |
| {YYYY-MM-DD HHMM} | {filename} | {N}% {Status} | {A-1, A-2… applied; A-3 deferred; A-4 waiting on user} |
```

Optional: keep a short **Pre-Flight Amendments** table (Finding → Resolution) for auditor skimming — still must match the rewritten steps.

### What not to do

- Do not invent new scope beyond the ledger + user input.
- Do not execute production code or "start implementing."
- Do not delete Revision History from prior updates.
- Do not leave duplicate conflicting instructions ("old: X" / "new: Y") in the same step.
- Do not introduce `file:///` links or developer home directory paths (`/home/...`) into the plan.

---

## Step 6: Coherence Gate

Before writing is considered done, verify:

- [ ] Every ledger row is `applied`, `deferred-to-execute`, or `blocked-on-user` (none left `pending` / `needs-evidence`)
- [ ] No `blocked-on-user` row was silently assumed
- [ ] Grep/read the plan: each applied amendment's key phrase or contract appears in the targeted step
- [ ] Frontmatter todos match atomic steps 1:1
- [ ] Every new/changed behavior has ≥1 Test Spec row (or explicit deferral noted)
- [ ] No step still contains the exact stale claim called out by Path to Green
- [ ] Probe templates exist for every Complex step
- [ ] Path & Link Hygiene: zero `file:///` URLs and zero `/home/...` developer paths in plan text; internal files are repo-relative (`docs/plans/template.md`) and external links are canonical Git web URLs

**If any box fails:** fix before Step 7. Do not hand a half-updated plan to `/pre-flight`.

---

## Step 7: Write Plan

1. Write the full updated `{slug}.plan.md` at its existing path.
2. Keep `{feature-slug}.plan.md` as the single source of truth (same path as `/create-plan`).

---

## Step 8: Present Summary

**In this conversation**, present:

1. Amendment Ledger (final statuses)
2. Diff-style summary: what changed per step/section (bullets, not necessarily a unified diff)
3. Open `blocked-on-user` / `defer-to-execute` items
4. Recommended next command:
   - Material plan changes → *"Plan updated. Run `/pre-flight` or `/converge-plan` to re-score toward ≥90%."*
   - Only deferrals and latest pre-flight already ≥90% → *"No material plan gaps left. Run `/execute-plan` when ready."*
   - Blocked on user decisions → *"Run `/grill-me` to lock decisions for {A-n…}, or reply directly and re-run `/update-plan` / `/converge-plan`."*

---

## Invocation styles

```
/update-plan
```

Uses newest plan + newest pre-flight in `.agents/plans/` (or `.cursor/plans/`).

```
/update-plan .agents/plans/{slug}.plan.md
```

Pinned plan path.

```
/update-plan

Decisions: use Postgres full-text search; drop Step 11 from scope; accept naive circuit-breaker trade-off.
```

User decisions applied as ledger rows (`user-decision` → immediate `plan-text`/`scope` once stated).

---

## Anti-Patterns

- Treating Path to Green as done without editing `.plan.md`
- Stacking "Amendment" prose on top of still-wrong pseudocode
- Re-running `/create-plan` for a handful of plan-text fixes
- Spawning a full 3-explorer create-plan panel for surgical edits
- Guessing `user-decision` items
- Marking contract/control-flow fixes as `defer-to-execute`
- Resetting unrelated completed todos
- Skipping Test Spec / frontmatter sync after scope changes
- Deliverable only in chat with no file write

## Pipeline position

| # | Skill | Actor |
| :--- | :--- | :--- |
| 1 | `/architect-bootstrap` | main agent (+ scan panel) |
| 2 | `/create-plan` | main agent (+ explorers) |
| 3 | `/grill-me` | main agent (interactive decision-tree interview) |
| 4 | `/pre-flight` | main agent (+ auditors, iterative) |
| 5 | `/update-plan` | **main agent** (+ conditional explorers) |
| 6 | `/converge-plan` | main agent (orchestrates pre-flight ⇄ update-plan) |
| 7 | `/execute-plan` | main agent (+ test specifier + executor) |
| 8 | `/codereview` | main agent (+ review panel) |
| 9 | `/open-mr` | main agent (+ CI & review monitor) |
| 10 | `/eop` | main agent (terminal) |
