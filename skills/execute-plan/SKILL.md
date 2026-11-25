---
name: execute-plan
description: >-
  Stage 6 of the agent pipeline. Executes an approved implementation plan
  using Red-First ATDD (Test Specifier Red harness -> Contract Lock -> Code Executor Green SUT),
  running safe-to-fail probes for Complex steps, and reconciling test results.
---

# Plan Execution

**Invoke via:** `/execute-plan`

**You are the coordinator.** Execute Steps 1–7 in order yourself. Main agent owns status, contract protection, and todo sync — see [Agent Pipeline Contract](../agent-pipeline/SKILL.md).

Execute via **Red-First ATDD Closed Loop**:
1. **Phase 4A (Red Harness):** Dispatch `execute-plan-tests` to author 4-layer acceptance scenarios and protocol drivers from Step 7, then prove tests fail for the expected domain reason (**RED**).
2. **Phase 4B (Contract Lock):** Seal test files by recording their paths and git SHAs. The code executor has zero permission to modify or relax tests.
3. **Phase 4C (Green Implementation):** Dispatch `execute-plan` to write production code until all locked acceptance tests pass (**GREEN**).
4. **Phase 4D (Micro-Batch Verification):** Verify tests are untouched, run the full verification suite, and sync todo status.

**Requires:** `{slug}.plan.md` and ideally a latest `*-Pre-Flight-Review_*.md` (soft gate).

**Pipeline:** `/pre-flight` → (`/update-plan` → `/pre-flight`)* | `/converge-plan` → **this** → `/codereview`

## Execution checklist

| Step | Who | Action | Gate |
| :--- | :--- | :--- | :--- |
| 1 | Main agent | **Context recall** (`AGENTS.md`, prior execution outcomes) | Prior context loaded |
| 2 | Main agent | Load plan + **latest** pre-flight report | Execution Context Package built |
| 3 | Main agent | Gate 0 (soft) | User informed; override OK |
| 4 | Test Specifier + Coder | **Red-First Phased Dispatch** (Red harness → Contract Lock → Green SUT) | All scenarios green & tests untouched |
| 5 | Main agent | **Reconciliation & Integrity Gate** — verify tests pass & test files unmutated | Pass/fail table + contract audit |
| 6 | Main agent | Sync plan frontmatter todos | Status table in chat |
| 7 | Main agent | Write outcome file & present summary | User sees todo progress + CTA |

---

## Step 1: Context Recall

Read `AGENTS.md` (workspace and user-level if present) and any prior `*-execution-outcome_*.md` for this plan slug.

---

## Step 2: Execution Context Package (by Reference)

Read from disk:
1. **Plan path** (`.plan.md`)
2. **Latest pre-flight report path** — newest `*-Pre-Flight-Review_*.md` by filename timestamp (if any exist)
3. Current todo statuses from plan frontmatter

```markdown
## Execution Context Package: {plan-slug}

### 1. Plan Identity
- Path: absolute path to `.plan.md` (read directly via file tools)
- Overview, workspace

### 2. Pre-Flight Status
- Latest report path + date
- Overall confidence, Ready/Caution/Stop
- Path to Green blockers (if any)
- Prior pre-flight reports summarized (if re-runs exist)

### 3. Execution State
- Todo list with current status (`pending`/`in_progress`/`completed`/`blocked`)
- Next actionable todo id

### 4. Prior Art
- Relevant `AGENTS.md` constraints from Step 1
- Prior execution outcomes for this plan

### 5. Mission
Implement approved plan step by step using Red-First ATDD. Update plan frontmatter todos. Spawn `probe-runner` only for Complex/probe steps. Propose commit messages. Do not skip verification.
```

**Gate:** Sections 1–3 complete before Step 4.

---

## Step 3: Gate 0 (Soft — No Pushback)

Using the **latest** pre-flight report:

| Condition | Main agent action |
| :--- | :--- |
| No pre-flight report | Recommend `/pre-flight`. **Do not refuse** — user may override to execute. |
| Confidence ≥ 90%, not Stop | State: *"Pre-flight passed at {N}%. Proceeding."* |
| Confidence < 90% or Stop | List blockers from Path to Green. Suggest **`/converge-plan`** or **`/update-plan`** → `/pre-flight`. **User explicit override → proceed anyway.** |
| User re-ran `/pre-flight` since last execute | Use newest report only — no lecture about prior runs. |

Iterative pre-flight is **expected**. Never block the user from running `/pre-flight` or `/execute-plan` again.

---

## Step 4: Red-First Phased Dispatch & Contract Lock

Follow Farley's 3 rules for AI execution:
1. **Red Before Green:** Run tests before writing implementation code; failure is the only proof the test detects defects.
2. **Anti-Tampering Contract:** Test files are immutable setpoints. Coding agents must not modify tests to make them pass.
3. **Small Verifiable Batches:** Step-by-step implementation against the locked test harness.

### Phase 4A: Test Specifier (`execute-plan-tests`)
1. Dispatch `execute-plan-tests` (`agents/execute-plan-tests.md`) with the Execution Context Package by reference.
2. The Test Specifier authors the 4-layer acceptance tests (Intent, DSL, Protocol Drivers, SUT targets) from plan Step 7.
3. **Red Failure Proof:** The Test Specifier runs the test suite and reports explicit failure output (**RED**):
   - Must fail for the expected domain reason (e.g., missing function, assertion mismatch, endpoint 404).
   - If tests fail due to syntax errors or broken imports, fix the test harness first.
   - If tests pass before implementation code is written, revise assertions until failing on the new requirement.

### Phase 4B: Contract Lock (Anti-Tampering Seal)
1. Record test file paths and git SHAs (`git hash-object` / `git diff`).
2. The test suite is now the **locked behavioral contract (Setpoint)**.

### Phase 4C: Code Executor (`execute-plan`)
1. Dispatch `execute-plan` (`agents/execute-plan.md`) with the Execution Context Package, sealed test paths, and failing test diagnostics.
2. **Anti-Tampering Rule Enforced:** The prompt strictly forbids editing, deleting, or relaxing any test files.
3. Executor iterates in small batches on production code until all locked acceptance tests turn **GREEN**.

### Phase 4D: Micro-Batch Verification
1. Verify `git diff --name-only` confirms sealed test files were untouched during Phase 4C.
2. Run full local verification (build + lint + tests).

---

## Step 5: Reconciliation & Contract Integrity Gate

After Phase 4 completes, the main agent verifies contract integrity:

| Check | Command | Expected |
| :--- | :--- | :--- |
| Contract Integrity | `git diff --name-only` / SHA compare on test files | Test files unchanged during coder phase |
| Test Execution | project test runner | 100% pass |
| Spec Coverage | test report vs plan Step 7 scenarios | every scenario covered |

### Reconciliation outcomes

| Outcome | Action |
| :--- | :--- |
| **All tests pass & contract intact** | Tests validate implementation cleanly. Proceed to Step 6. |
| **Tests fail — implementation bug** | Re-dispatch `execute-plan` worker to fix implementation code (main agent MUST NOT patch code directly). |
| **Test was tampered with by executor** | Revert test mutations; re-dispatch executor with strict warning to fix production code only. |
| **Spec rows uncovered** | Dispatch `execute-plan-tests` to add missing scenarios before proceeding. |

---

## Step 6: Sync Plan Todos

Ensure `.plan.md` frontmatter reflects actual todo statuses from execution.

---

## Step 7: Write Outcome & Present

1. Write `.agents/plans/{slug}-execution-outcome_YYYY-MM-DD_HHMM.md` (or `.cursor/plans/...`) with todo table, commits, blockers, next steps, and reconciliation results.
2. Present **Todo status table** in this conversation.
3. Present **Reconciliation summary** — tests passed/failed/diverged, spec coverage, contract integrity.
4. End with:
   - All completed + tests pass → *"Branch out and run `/codereview`."*
   - Blocked on probes → *"Run `/execute-plan` again when probes complete."*
   - Reconciliation failures unresolved → *"Fix divergences, then run `/codereview`."*
   - Plan gaps found → *"Run `/converge-plan` or `/update-plan` → `/pre-flight`, then `/execute-plan` again."*

---

## Anti-Patterns

- Main agent patching production code directly instead of delegating to `execute-plan`
- Ignoring latest pre-flight report when a newer one exists
- Refusing execution because an older pre-flight failed (use latest only)
- Letting the code executor modify or relax sealed test files
- Skipping Red Failure Proof before writing production code
- Skipping plan frontmatter todo updates
- Skipping reconciliation when tests fail

## Pipeline position

| # | Skill | Actor |
| :--- | :--- | :--- |
| 1 | `/architect-bootstrap` | main agent (+ scan panel) |
| 2 | `/create-plan` | main agent (+ evidence explorers) |
| 3 | `/pre-flight` | main agent (+ `pre-flight` auditors, iterative) |
| 4 | `/update-plan` | main agent (+ conditional explorers) |
| 5 | `/converge-plan` | main agent (orchestrates pre-flight ⇄ update-plan) |
| 6 | `/execute-plan` | **main agent** (+ `execute-plan-tests` + `execute-plan`) |
| 7 | `/codereview` | main agent (+ specialized review panel) |
| 8 | `/open-mr` | main agent (+ CI & review monitor) |
| 9 | `/eop` | main agent (terminal) |
