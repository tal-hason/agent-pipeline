---
name: architect-bootstrap
description: >-
  Stage 1 of the agent pipeline. Scans workspace boundaries, root manifests,
  and architecture to build an architect brief and visual canvas with situational awareness.
  Use when starting a new project, exploring an unfamiliar workspace, or defining the
  'where and what' before planning.
---

# Paired Systems Architect

**Invoke via:** `/architect-bootstrap`

**You are the coordinator.** Execute Steps 1–8 in order yourself (Step **2.5** is mandatory when intent is vague). Main agent owns the deliverable — see [Agent Pipeline Contract](../agent-pipeline/SKILL.md).

Workers: **specialized scanners** in Step 4 (`readonly`) — **evidence only**. They return markdown reports; they **never** write files.

**Ideation** runs in the **main agent** in Step 2.5 — interactive, user-facing. Do not dispatch interactive ideation as background subagents.

**You alone** merge findings, **write `architect-brief.canvas.tsx`**, and present the brief in chat.

**Pipeline:** [Agent Pipeline Contract](../agent-pipeline/SKILL.md).

**Two lanes:** **WHERE** (workspace scan → canvas) + **WHAT** (ideation when intent is unclear). **Next:** `/create-plan` (HOW) — carries brainstorm requirements when present.

## Execution checklist

| Step | Who | Action | Gate |
| :--- | :--- | :--- | :--- |
| 1 | Main agent | **Context recall** (`AGENTS.md`, prior digests, brainstorms) | Prior context loaded |
| 2 | Main agent | **Bootstrap Context Brief** | Brief draft (sections 1–7) |
| 2.5 | Main agent | **Intent gate + ideation lane** | User intent clear **or** requirements doc linked |
| 3 | Main agent | **Skills + Scanner triage** | Panel table published |
| 4 | Scanners (parallel) | Specialized subagents (`readonly`) | All returned — reports only |
| 5 | Main agent | Merge scan + compatibility/prior-art synthesis | Situational Awareness table ready |
| 6 | Main agent | **Write `architect-brief.canvas.tsx`** | Canvas on disk — **not subagents** |
| 7 | Main agent | Present situational awareness in chat | User sees brief + canvas path |
| 8 | Main agent | Verify artifacts persisted on disk + CTA | Ready for `/create-plan` |

---

## Step 1: Context Recall

Read `AGENTS.md` (workspace and user-level if present), any existing `docs/brainstorms/*`, `docs/solutions/*`, and recent `.agents/digests/*` before scanning.

Carry relevant findings into Step 2 Prior Art.

---

## Step 2: Bootstrap Context Brief

**Main agent only.** Apply [probe-before-assume](../probe-before-assume/SKILL.md): read `README`, `AGENTS.md`, root manifests — do not guess stack or boundaries.

```markdown
## Bootstrap Context Brief: {workspace-slug}

### 1. User Intent
### 2. Starting Hypothesis
### 3. Scan Questions (numbered — scanners must answer each)
### 4. Prior Art (context hits from Step 1)
### 5. Workspace Type Signals
- Primary kind: code repo | docs/coordination | platform/GitOps | monorepo | multi-root
- Build surfaces detected: (package.json, go.mod, pyproject.toml, Makefile, Dockerfile, GitHub Actions, GitLab CI, etc.)
- Agent-facing docs: README quality, AGENTS.md, rules presence

### 6. Initial Cynefin Read
- Domain for the stated intent: Clear | Complicated | Complex | Chaotic | Disorder
- If Disorder: list clarifying questions before heavy scanning

### 7. Requirements Anchor (if any)
- Path to `docs/brainstorms/*-requirements.{md,html}` or `docs/ideation/*` artifact
- One-paragraph scope summary for scanners (from brainstorm or user)
- Link to `STRATEGY.md` at repo root if it exists
```

**Gate:** Sections 1–3 and 5 complete before Step 2.5.

---

## Step 2.5: Intent Gate & Ideation Lane

**Main agent only.** Bootstrap answers **WHERE** (workspace reality). Ideation answers **WHAT** (scope, behavior, success criteria) when intent is not yet plan-ready.

### Route (pick one)

| Signal | Route | Action |
| :--- | :--- | :--- |
| **Clear intent** — acceptance criteria, bounded scope, or explicit "scan only" | **Skip ideation** | Finalize brief §1–2; proceed to Step 3 |
| **Vague intent** — "improve X", "explore", Disorder, missing §1 | **`brainstorm`** | Run interactive Q&A (one question per turn). Write or update `docs/brainstorms/{slug}-requirements.md` when durable handoff needed |
| **No chosen direction** — "what should we build", "give me ideas" | **`ideate` → `brainstorm`** | Produce ranked options in `docs/ideation/`; user picks one; brainstorm defines it |
| **`STRATEGY.md` missing** on product/code repo with greenfield intent | **Flag only** | Note in brief + canvas; suggest strategy session — do **not** block workspace scan |
| **Existing brainstorm/ideation doc** in repo | **Resume** | Read artifact; confirm with user; fold into brief §7 |

### Ideation rules

1. **Interactive ideation stays in main agent** — do not Task-dispatch interactive ideation subagents.
2. **Workspace-only bootstrap** — user says "bootstrap this repo" with no feature → skip brainstorm; scan only.
3. **Lightweight when clear** — if requirements already clear, confirm in 2–3 lines and skip long Q&A.
4. **After ideation** — update brief §1 User Intent, §2 Hypothesis, §7 Requirements Anchor; scanners validate against §7 in Step 4.

**Gate:** §1 is actionable **or** §7 links a requirements doc before Step 3.

---

## Step 3: Skills + Scanner Triage

**Main agent only.** Read applicable skills **before** dispatching scanners. Publish the scanner panel in chat.

### Skills (read and inject into prompts)

| Skill | Path | When to read | Inject into |
| :--- | :--- | :--- | :--- |
| **Probe before assume** | `skills/probe-before-assume/SKILL.md` | **Always** | Brief discipline + every scanner prompt |
| **Cynefin sense-making** | `skills/cynefin-sense-making/SKILL.md` | **Always** | Brief §6 + Step 2.5 routing |
| **Platform design review** | `skills/platform-design-review/SKILL.md` | Platform/API/IaC/SDK/GitOps surface detected | Scanner **C** + canvas Platform Economics section |

### Core scanners (always dispatch)

| ID | `subagent_type` | Lens |
| :--- | :--- | :--- |
| **A** | `ce-repo-research-analyst` | Structure, conventions, docs, call chains, boundaries (`very thorough`) |
| **B** | `explore` | Stack, CI/CD, dependencies, entrypoints (`medium` or `very thorough`) |
| **C** | `ce-architecture-strategist` | Hexagonal boundaries, coupling, platform economics signals, long-term risks |

### Conditional scanners (dispatch when signal matches)

| ID | `subagent_type` | Trigger |
| :--- | :--- | :--- |
| **D** | `compatibility-scan-review` | Code repo with README + build surface — deterministic agent-compat score |
| **E** | `startup-review` | Same as D — cold-start path reliability |
| **F** | `validation-review` | Same as D — cost of verifying a small change |
| **G** | `docs-reliability-review` | Same as D — docs vs reality check |
| **H** | `ce-git-history-analyzer` | `.git` present + code repo (not docs-only) |
| **I** | `ce-learnings-researcher` | `docs/solutions/` exists OR institutional-knowledge request |
| **J** | `ce-web-researcher` | External prior art, standards, or market context requested |

**Agent Compatibility lane:** When **D** triggers, dispatch **D + E + F + G** together (four separate tasks — do not collapse). Synthesize **Agent Compatibility Score** (`deterministic * 0.7 + workflow_avg * 0.3`).

**Panel limits:** Always dispatch **A + B + C**. Cap **10 parallel** tasks per message. If over cap: **A → B → C** (never drop) → **D–G bundle** → **H** → **I**.

**Model policy:** Core scanners (A–C, H–I) use **thinking-tier** models only, using a distinct model from the coordinator when supported.

---

## Step 4: Parallel Workspace Scan (WHERE)

**WAIT** for all scanners. **Subagents do NOT write Canvas or any files.**

Each prompt MUST include:

1. Full **Bootstrap Context Brief** (all sections from Step 2)
2. **Skill addendum** — pasted checklist from Step 3 (probe, platform, etc. as applicable)
3. **Mission:** return markdown scan report only — **do not write files, do not create canvas**
4. **Output contract:** tech stack, boundaries, files validated, gaps/risks, answers to Scan Questions; **validate brief §7 requirements against codebase** when present; compatibility agents use their plain-text score format.

`readonly: true` on every scanner.

---

## Step 5: Merge + Prior-Art Synthesis (WHERE + WHAT)

**Main agent only.**

1. Merge scanner reports (union; flag contradictions).
2. **Agent Compatibility synthesis** — if D–G ran, compute combined score (`deterministic * 0.7 + workflow_avg * 0.3`) and merge top fixes into gaps table.
3. **Institutional learnings** — if **I** ran, surface applicable `docs/solutions/` hits in Prior Art for `/create-plan`.
4. Build **Situational Awareness** data for Canvas (tables, stats, gaps).

---

## Step 6: Write Canvas (main agent only)

**Only the main agent** writes the canvas file. Scanners never reach this step.

1. Follow `references/architect-brief-canvas.md` structure.
2. Write to: `canvases/architect-brief.canvas.tsx` (or `.agents/canvases/architect-brief.canvas.tsx`).
3. Canvas sections:
   - Header stats (include **Agent Compatibility Score** when D–G ran)
   - **Intent & scope** — user intent, link to requirements doc (`docs/brainstorms/`), `STRATEGY.md` one-liner if present
   - Situational Awareness table
   - Key boundaries (Ports & Adapters)
   - Gaps / risks (union; compatibility top fixes merged here)
   - **Platform Economics** table (when platform-design-review skill applied)
   - Pipeline footer (`bootstrap → create-plan → pre-flight → …`)

**Gate:** File exists on disk before Step 7. Chat-only output is not sufficient.

---

## Step 7: Present in Chat

Summarize: **intent & scope** (or requirements doc path), tech stack, boundaries, top risks, agent compatibility (if run), **Initial Cynefin read**, **path to canvas**. Full brief in conversation — not only in the file.

---

## Step 8: Persist & Hand Off

Confirm canvas file and any `docs/brainstorms/*` requirements doc are saved on disk.

End with one of:
- *"Bootstrap complete. Requirements at `{path}`. Run `/create-plan` when ready."*
- *"Bootstrap complete (workspace scan only). Run `/create-plan` when ready."*

---

## Anti-Patterns

- Letting scanners or any subagent write `.canvas.tsx`
- 3× generic `explore` with identical lens (use specialized panel)
- Scanners without full Bootstrap Context Brief + skill addendum
- Collapsing D–G compatibility checks into one prompt
- Chat-only deliverable with no canvas file
- Guessing workspace type without reading manifests/README
- Heavy scanning while User Intent is still Disorder with no §7 anchor
- Skipping Step 2.5 when user presents a vague feature request

## Pipeline position

| # | Skill | Actor |
| :--- | :--- | :--- |
| 1 | `/architect-bootstrap` | **main agent** (Canvas author) + specialized readonly scanners |
| 2 | `/create-plan` | main agent (+ evidence explorers) |
| 3 | `/grill-me` | main agent (interactive decision-tree interview) |
| 4 | `/pre-flight` | main agent (+ `pre-flight` auditors, iterative) |
| 5 | `/update-plan` | main agent (+ conditional explorers) |
| 6 | `/converge-plan` | main agent (orchestrates pre-flight ⇄ update-plan) |
| 7 | `/execute-plan` | main agent (+ test specifier + executor) |
| 8 | `/codereview` | main agent (+ specialized review panel) |
| 9 | `/open-mr` | main agent (+ CI & review monitor) |
| 10 | `/eop` | main agent (terminal) |
