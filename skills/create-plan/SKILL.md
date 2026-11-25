---
name: create-plan
description: >-
  Stage 2 of the agent pipeline. Dispatches evidence explorers to investigate
  the codebase and drafts an atomic, evidence-backed implementation plan (.plan.md) with
  Cynefin classifications, 4-layer ATDD test specifications, and probe designs.
  Use when designing the implementation strategy ('how') before writing code.
---

# Technical Plan

**Invoke via:** `/create-plan`

**You are the coordinator.** Execute Steps 1–8 in order yourself. Main agent owns the deliverable — see [Agent Pipeline Contract](../agent-pipeline/SKILL.md).

The only subagents you may spawn are the **3 evidence explorers** in Step 3 (`readonly: true`). You merge their output and author the final plan in **this** conversation.

**Pipeline:** See [Agent Pipeline Contract](../agent-pipeline/SKILL.md). **Prior stage:** `/architect-bootstrap` (optional — carry Canvas brief into Step 2).

## Execution checklist

| Step | Who | Action | Gate before next step |
| :--- | :--- | :--- | :--- |
| 1 | Main agent | **Context recall** (`AGENTS.md`, canvas brief, brainstorm docs) | Prior context loaded |
| 2 | Main agent | Write **Exploration Design Brief** + Mermaid map | Brief complete — ready to attach to explorer prompts |
| 3 | 3 explorers (parallel) | Dispatch A, B, C (`ce-repo-research-analyst`, `explore`, `ce-correctness-reviewer`) | **All 3 completed** (or A+C if 2-model fallback) |
| 4 | Main agent | Merge evidence + validation checklist | No unresolved contradictions (or user resolved) |
| 5 | Main agent | Implementation strategy | Strategy cites merged evidence |
| 6 | Main agent | Atomic execution steps + probe templates | Every affected file has a step |
| 7 | Main agent | Verification & 4-layer ATDD Test Specification | Every affected file maps to ≥1 scenario |
| 8 | Main agent | Write `.plan.md` & present full plan | User can review full plan in this chat |

**Next in pipeline (after user approves):** `/pre-flight` or `/converge-plan` → `/execute-plan` → `/codereview` → `/open-mr` → `/eop`

---

## Step 1: Context Recall

Read `AGENTS.md` (workspace and user-level if present), any recent `.agents/digests/*`, and prior plans if relevant.

If `/architect-bootstrap` ran earlier in this session, load the situational awareness brief / canvas summary into Step 2. If bootstrap produced a requirements anchor (`docs/brainstorms/*-requirements.md` or brief §7), treat it as the primary input for §1 Problem & Goals — do not re-invent scope.

---

## Step 2: Exploration Design Brief

**Main agent only** — author this **before** dispatching explorers. Do not forward the raw user prompt as the exploration input.

Synthesize the user request, context hits (Step 1), and your architectural read into a **Summarized Design Plan for Exploration**. Explorers use this brief to validate, refute, and enrich — not to infer intent from a one-liner.

### Required brief structure

```markdown
## Exploration Design Brief: {feature-slug}

### 1. Problem & Goals
- What problem are we solving? What does success look like?
- In scope / out of scope (explicit non-goals)

### 2. Constraints
- Technical, security, compatibility, or policy constraints
- Relevant prior art or architecture docs (from Step 1)

### 3. Proposed Approach (pre-evidence)
- High-level strategy in 3–5 sentences — hypothesis, not fact
- Key design decisions still to validate

### 4. Suspected Impact
- Components, services, repos, directories likely touched
- Suspected files (initial hypothesis — explorers confirm or reject)
- Downstream consumers at risk

### 5. Open Questions for Explorers
- Numbered list of assumptions that MUST be verified in code
- e.g. "Does `FooService` already expose X?", "Is there an existing pattern for Y?"

### 6. Architecture Map (Mermaid)
- System context (`graph TD` or `sequenceDiagram`)
- Data flow: enter → through → exit
- Impact radius: downstream consumers highlighted

### 7. Exploration Success Criteria
- What evidence must explorers return before you can write atomic steps?
```

**Gate:** The brief must be complete (all 7 sections) before Step 3. You may show it in chat before dispatching.

---

## Step 3: Evidence-Gathering Exploration

**WAIT:** Do not start Step 4 until all dispatched explorers have returned.

Dispatch **3 explorers** in parallel using distinct reasoning models where supported. `readonly: true` on each explorer.

| Explorer | `subagent_type` | Lens |
| :--- | :--- | :--- |
| **A — Deep trace** | `ce-repo-research-analyst` | Repo structure, conventions, full-file reads, call chains, transitive consumers (`very thorough`) |
| **B — Reasoned scout** | `explore` | Hypothesis-driven discovery, entrypoints, suspected-area sweep (`medium` / `very thorough`) |
| **C — Probe & correctness** | `ce-correctness-reviewer` | Challenge assumptions, edge cases, contradictions; **read-only** probe checks |

### Shared prompt payload (all explorers)

Attach the **full Exploration Design Brief** from Step 2 as the primary input. Do not substitute a shortened user prompt.

Each prompt MUST include:

1. **Exploration Design Brief** — entire document from Step 2 (all 7 sections)
2. **Mission statement** — *"Validate, refute, and enrich this design brief with codebase evidence. Do not author the final plan."*
3. **Communication Contract** — strict telegraphic style (caveman), zero conversational filler or greetings, output structured evidence report directly
4. **Output contract** — return only the structured evidence report (format below); no plan authoring

### Role-specific instructions

- **A:** Prioritize depth over breadth. Trace callers and transitive consumers. Read full files, not just signatures.
- **B:** Prioritize coverage. Validate or refute the hypothesis map. Find files A might miss.
- **C:** Prioritize falsification. List assumptions that could be wrong; cite read-only evidence (test files, configs, command output) that confirms or refutes them.

### Required evidence report format

Each explorer returns these sections:
1. **Affected Files:** Path, Evidence (function/import/line), `@ai-shebang` constraints
2. **Dependencies:** External libs, internal modules, transitive consumers
3. **Existing Patterns:** Naming, error handling, logging/observability, testing
4. **Unknowns & Contradictions:** Missing files, ambiguous interfaces, conflicting patterns
5. **Cynefin Pre-Classification:** Clear / Complicated / Complex + rationale

---

## Step 4: Merge Exploration Evidence

**Main agent only** — merge the three reports:

| Merge rule | Rule |
| :--- | :--- |
| Affected files | UNION — any explorer listing a file keeps it |
| Dependencies | UNION all consumers |
| Patterns | Agreement = established; disagreement = flag |
| Unknowns | UNION all |
| Contradictions | Factual disagreements → **STOP**, present to user, wait for resolution |
| Cynefin | Take the **more complex** classification when explorers disagree |

### Evidence validation (required before Step 5)

- [ ] Every affected file was read by ≥1 explorer (not assumed)
- [ ] Every `@ai-shebang` header captured
- [ ] Every unknown listed explicitly
- [ ] No unresolved contradictions (or user chose how to resolve)

**Gate:** If contradictions remain unresolved, do not proceed to Steps 5–8.

### Explorer agreement matrix (include in final plan)

| Topic | A | B | C | Resolution |
| :--- | :--- | :--- | :--- | :--- |
| *(file, pattern, or assumption)* | agree / disagree / N/A | … | … | merged decision |

---

## Step 5: Implementation Strategy

Using merged evidence only:
1. **Pattern selection** — match existing codebase patterns (cite explorers).
2. **Breaking changes** — list consumers at risk with file evidence.
3. **Complexity per area** — merged Cynefin tags.

---

## Step 6: Atomic Execution Steps

Numbered steps. Each must be:
- **Isolated** — implementable and verifiable on its own where possible
- **Specific** — e.g. "Update `UserService.ts` to handle null email", not "Fix bug"
- **Evidence-linked** — cites file + `@ai-shebang` from Step 4
- **Cynefin-tagged** — Clear / Complicated / Complex
- **Verification-defined** — build, test, or lint command that proves done

**Complex steps** must include an inline probe template (`rules/plan-probes.md`):
- Hypothesis, Method (safe-to-fail), Sensing, Acceptance criteria
- Mark: `**Cynefin: Complex -- probe required**`

**Gate:** Every file in the merged affected-files list must map to ≥1 step.

---

## Step 7: Verification & Test Specification (Farley 4-Layer ATDD & Executable Specifications)

This section feeds **two consumers**: the executor (verification gates) and the **test specifier** worker (`execute-plan-tests`) in `/execute-plan`. Prose specifications drift; specifications MUST be executable before code is written. Structure verification using Dave Farley's 4-layer acceptance test model:

```
+-----------------------------------------------------------+
| Layer 1: Intent (Given / When / Then - Domain Language)   |
+-----------------------------------------------------------+
| Layer 2: Domain DSL (Language of the Problem Domain)      |
+-----------------------------------------------------------+
| Layer 3: Protocol Drivers (HTTP, CLI, gRPC, DB Adapters)  |
+-----------------------------------------------------------+
| Layer 4: System Under Test (SUT)                          |
+-----------------------------------------------------------+
```

### 7a. Executable Scenarios (4-Layer Acceptance Specifications)

```markdown
| ID | User / Domain Scenario (Intent) | DSL Operation (Domain Level) | Protocol Driver / Adapter | SUT Target | Category |
| :--- | :--- | :--- | :--- | :--- | :--- |
| T-1 | Given [state], When [action], Then [outcome] | dsl.doDomainAction(...) | HttpDriver / MemoryDriver | target module/endpoint | acceptance / unit |
| T-2 | Given [state], When [invalid action], Then [domain error] | dsl.doInvalidAction(...) | HttpDriver / MemoryDriver | target module/endpoint | acceptance / unit |
| T-3 | Given [boundary condition], When [action], Then [handled] | dsl.checkBoundary(...) | HttpDriver / MemoryDriver | target module/endpoint | acceptance |
```

Rules for executable specifications:
- **Executable Over Prose**: Minimum 3 scenarios per major change (success, failure/error, edge/boundary).
- **Zero Protocol Leaks in Intent**: Layer 1 must contain ZERO HTTP status codes, JSON payloads, or database column names. Express pure business intent.
- **Seam Isolation**: Protocol Drivers (Layer 3) isolate tests from infrastructure volatility so transport changes do not break test specifications.
- **Baseline Fitness Tests**: When deferring non-functional scaling (concurrency, caching, rate limiting), specify an automated fitness test to guard the boundary assumption.
- For refactors: include "behavior unchanged" regression rows.

### 7b. Test Infrastructure & Driver Notes
- Existing test framework / runner (from explorer evidence)
- Test file and driver directory patterns (`test/dsl/`, `test/drivers/`)
- Mock/stub/test-double patterns for driving and driven ports
- Test execution command: `npm test`, `pytest`, `go test ./...`

### 7c. Evidence Coverage Gate
- Confirm Step 6 covers all explored files.
- Confirm every affected file maps to ≥1 test specification row.

---

## Step 8: Write and Present the Plan

### 8a. Write file

Path: `.agents/plans/{feature-slug}.plan.md` (or `.cursor/plans/{feature-slug}.plan.md`).

YAML frontmatter:

```yaml
---
name: feature-slug
overview: One-line summary
issue: null # optional issue/ticket reference (e.g. #123)
todos:
  - id: step-1
    content: "[Clear] Specific atomic step with verification"
    status: pending
  - id: step-2
    content: "[Complex] Step summary — probe required"
    status: pending
isProject: false
---
```

### 8b. Plan body (required sections)
1. Mermaid architecture diagram (from Step 2, updated if merge changed scope)
2. Merged evidence summary
3. Explorer agreement matrix (from Step 4)
4. Implementation strategy
5. Atomic execution steps (with probe templates for Complex)
6. Verification & Test Specification (from Step 7 — includes test spec table)

### 8c. Mandatory Path & Link Hygiene
Plans may be committed to repositories and reviewed across environments. Strict portability is mandatory:
- **Zero `file:///` URLs** anywhere in the plan document, frontmatter, tables, or commit messages.
- **Zero Developer Home Paths** (`/home/...`, `~/...`, or workstation-specific directories).
- **Repo-Relative Paths for Target Files** (e.g. `docs/plans/template.md`, `src/services/config.ts`).
- **Canonical HTTPS Git URLs** for cross-repo references (e.g. `https://github.com/<org>/<repo>/blob/<branch>/<path>`).

### 8d. Present in conversation
1. **Paste the full plan in this conversation** — not only on disk, not only in subagent output.
2. End with: *"Plan written. Review the plan above. When approved, run `/pre-flight` or `/converge-plan`."*

---

## Anti-Patterns

- Forwarding the raw user prompt to explorers instead of the Exploration Design Brief
- Committing developer home paths (`/home/...`) or `file:///` URLs into plans
- Starting Step 4 before all explorers finish
- Building on assumptions instead of explorer evidence
- One explorer, or same `subagent_type` / model twice
- Fast/non-thinking models for exploration or probe validation
- Skipping `@ai-shebang` reads
- Discarding a finding because only one explorer found it
- Vague atomic steps ("Fix bug", "Update module")
- Complex steps without probe templates
- Resolving contradictions silently without user visibility
- Leaving the deliverable only in subagent context or only as a file path

## Pipeline position

| # | Skill | Actor |
| :--- | :--- | :--- |
| 1 | `/architect-bootstrap` | main agent (+ scan panel) |
| 2 | `/create-plan` | **main agent** (+ explorer subagents in Step 3 only) |
| 3 | `/pre-flight` | main agent (+ `pre-flight` auditors) |
| 4 | `/update-plan` | main agent (+ conditional explorers) |
| 5 | `/converge-plan` | main agent (orchestrates pre-flight ⇄ update-plan) |
| 6 | `/execute-plan` | main agent (+ test specifier + executor) |
| 7 | `/codereview` | main agent (+ specialized review panel) |
| 8 | `/open-mr` | main agent (+ CI & review monitor) |
| 9 | `/eop` | main agent (terminal) |
