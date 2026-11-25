# Agent Pipeline

A complete, multi-harness plugin for **Antigravity (`agy`)**, **Cursor**, **Claude Code**, and **Codex** that turns a vague idea into reviewed, shipped, and documented code through ten structured stages.

```mermaid
flowchart TD
    subgraph P1 ["Phase 1 · Sense, Design & Align (WHERE · WHAT · HOW)"]
        S1(["1 · /architect-bootstrap — Scan, Ideate & Canvas Brief"])
        S2["2 · /create-plan — 3x Explorers, Plan & 4-Layer ATDD Spec"]
        S3(["3 · /grill-me — Decision-Tree Interview (1 Q/Turn + Recommended)"])
    end

    subgraph P2 ["Phase 2 · Readiness & Convergence (Setpoint Audit)"]
        S4{"4 · /pre-flight — 3x Auditors (≥ 90% Gate)"}
        S5["5 · /update-plan — Amendment Ledger & Coherence Gate"]
        S6[["6 · /converge-plan — Auto Loop (Max 3 Rounds, Stall Guard)"]]
    end

    subgraph P3 ["Phase 3 · Closed-Loop ATDD Execution & Review"]
        S7["7 · /execute-plan — Red Harness → Contract Lock → Green SUT"]
        S8{"8 · /codereview — R1–R6 Security & Architecture Panel"}
    end

    subgraph P4 ["Phase 4 · Ship & Compound Knowledge"]
        S9["9 · /open-mr — PR/MR Creation, CI & Review Monitor"]
        S10(["10 · /eop — Docs, AGENTS.md Learnings & Session Digest"])
    end

    S1 -->|"Canvas + Scope"| S2
    S2 -->|"Draft .plan.md"| S3
    S2 -.->|"Skip if Trivial"| S4
    S3 -->|"Decisions Locked"| S4
    S3 -.->|"Auto Converge"| S6
    S4 -->|"< 90% · Path to Green"| S5
    S5 -->|"Re-score Plan"| S4
    S6 <-->|"Orchestrates 4 ⇄ 5"| S4
    S4 -.->|"User-Decision Gaps"| S3
    S6 -.->|"Pause / Stall on Decisions"| S3
    S4 -->|"Ready (≥ 90%) or Override"| S7
    S6 -->|"Converged (≥ 90%)"| S7
    S7 -->|"Sealed Tests Green"| S8
    S8 -.->|"Fix Findings"| S7
    S8 -.->|"Plan-Level Gap"| S5
    S8 -->|"Clean Verdict"| S9
    S9 -->|"Safe Gates Merged"| S10

    classDef entryExit fill:#0f172a,stroke:#38bdf8,stroke-width:2.5px,color:#f8fafc
    classDef interview fill:#3b0764,stroke:#e879f9,stroke-width:2.5px,color:#fdf4ff
    classDef action fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#f8fafc
    classDef gate fill:#451a03,stroke:#f59e0b,stroke-width:2.5px,color:#fef3c7
    classDef loopNode fill:#064e3b,stroke:#34d399,stroke-width:2.5px,color:#ecfdf5

    class S1,S10 entryExit
    class S3 interview
    class S2,S5,S7,S9 action
    class S4,S8 gate
    class S6 loopNode

    style P1 fill:transparent,stroke:#64748b,stroke-width:1.5px,stroke-dasharray: 5 5
    style P2 fill:transparent,stroke:#64748b,stroke-width:1.5px,stroke-dasharray: 5 5
    style P3 fill:transparent,stroke:#64748b,stroke-width:1.5px,stroke-dasharray: 5 5
    style P4 fill:transparent,stroke:#64748b,stroke-width:1.5px,stroke-dasharray: 5 5
```

## What's in this plugin

One repository root packaged as a multi-harness plugin bundle (`agent-pipeline`):

```
agent-pipeline/
├── plugin.json                            # Antigravity root plugin manifest
├── .cursor-plugin/
│   ├── plugin.json                        # Cursor plugin manifest
│   └── marketplace.json                   # Cursor marketplace catalog entry
├── .claude-plugin/
│   ├── plugin.json                        # Claude Code plugin manifest
│   └── marketplace.json                   # Claude Code marketplace catalog entry
├── .codex-plugin/
│   └── plugin.json                        # Codex plugin manifest
├── .cursor/
│   └── hooks.json                         # Cursor lifecycle hooks (stop + afterFileEdit)
├── assets/
│   └── logo.svg                           # Plugin icon
├── skills/                                # 16 modular skills (progressive disclosure)
│   ├── agent-pipeline/SKILL.md            # Canonical 10-stage pipeline orchestrator
│   ├── architect-bootstrap/SKILL.md       # Stage 1: workspace scan + canvas brief
│   ├── create-plan/SKILL.md               # Stage 2: evidence-backed plan + 4-layer ATDD spec
│   ├── grill-me/SKILL.md                  # Stage 3: decision-tree alignment interview
│   ├── pre-flight/SKILL.md                # Stage 4: confidence audit (iterative, ≥90% gate)
│   ├── update-plan/SKILL.md               # Stage 5: surgical plan amendment + Coherence Gate
│   ├── converge-plan/SKILL.md             # Stage 6: automated pre-flight ⇄ update-plan loop
│   ├── execute-plan/SKILL.md              # Stage 7: Red-First ATDD implementation
│   ├── codereview/SKILL.md                # Stage 8: multi-reviewer panel
│   ├── open-mr/SKILL.md                   # Stage 9: PR/MR creation + CI & review monitor
│   ├── eop/SKILL.md                       # Stage 10: session digest + AGENTS.md learnings
│   ├── uiux-expert/SKILL.md               # Standalone: UI/UX design orchestrator
│   ├── generate-shebangs/SKILL.md         # Standalone: @ai-rules header generator
│   ├── probe-before-assume/SKILL.md       # Foundational: evidence-first behavior
│   ├── cynefin-sense-making/SKILL.md      # Foundational: Cynefin domain classification
│   └── platform-design-review/SKILL.md    # Foundational: platform review (6 tests + 7 C's)
├── agents/                                # 69 worker & specialist subagent definitions
│   ├── codereview.md                      # Core: Principal code reviewer (R1)
│   ├── pre-flight.md                      # Core: Cynefin confidence auditor
│   ├── execute-plan.md                    # Core: Plan executor (production code)
│   ├── execute-plan-tests.md              # Core: Test Specifier (Red-First ATDD)
│   ├── probe-runner.md                    # Core: Safe-to-fail experiment runner
│   ├── uiux-expert.md                     # Core: UI/UX design orchestrator
│   ├── generate-shebangs.md               # Core: @ai-rules header generator
│   ├── ce-*.md                            # 51 Compound Engineering specialists & scouts
│   ├── *-review.md / ci-watcher.md / ...  # 9 compatibility, CI, learning, & code-review agents
│   └── explore.md / general-purpose.md    # 2 pipeline built-in aliases
├── rules/                                 # 14 rules (AGENTS.md + 13 modular rules)
│   ├── AGENTS.md                          # Pipeline contract & cross-session learnings
│   ├── persona.md                         # Systems Architect persona
│   ├── orchestrator-mandate.md            # Main-agent delegation of code mutations
│   ├── cynefin.md                         # Cynefin sense-making framework
│   ├── kiss.md                            # KISS problem-solving principle
│   ├── ai-shebang.md                      # @ai-rules header enforcement
│   ├── ai-rules.md                        # Prompt engineering standards
│   ├── codebase-workflow.md               # Code conventions + context gathering
│   ├── refactor.md                        # Pre-refactor documentation
│   ├── microservices.md                   # Distributed systems patterns
│   ├── platform-strategy.md               # Platform design principles (7 C's)
│   ├── platform-economics.md              # Double Pyramid economics model
│   ├── platform-abstraction-quality.md    # Abstraction vs illusion discipline
│   └── plan-probes.md                     # Complex step probe templates
├── hooks/
│   └── codereview-reminder.sh             # Remind to run /codereview when files changed
├── references/
│   └── architect-brief-canvas.md          # Canvas structure for /architect-bootstrap
├── docs/
│   └── pipeline-howto.md                  # Team onboarding & usage guide
└── LICENSE                                # MIT License
```

## Installation

### Antigravity (`agy`)

Install directly from GitHub:

```bash
agy plugin install https://github.com/tal-hason/agent-pipeline
```

Validate or manage locally:

```bash
agy plugin list
agy plugin validate https://github.com/tal-hason/agent-pipeline --strict
```

### Cursor

Add `https://github.com/tal-hason/agent-pipeline` in **Cursor Settings → Plugins / Marketplaces**, or clone directly into your local Cursor plugins folder:

```bash
git clone https://github.com/tal-hason/agent-pipeline.git ~/.cursor/plugins/local/agent-pipeline
```

### Claude Code

Register the marketplace and install the plugin:

```text
/plugin marketplace add tal-hason/agent-pipeline
/plugin install agent-pipeline@agent-pipeline
```

### Codex / Manual Project-Level

Clone the repository and copy `skills/`, `agents/`, and `rules/` into a project's `.agents/` or `.cursor/` directory:

```bash
git clone https://github.com/tal-hason/agent-pipeline.git
mkdir -p /path/to/project/.agents
cp -r agent-pipeline/{skills,agents,rules} /path/to/project/.agents/
```

## How the pipeline works

See [`docs/pipeline-howto.md`](docs/pipeline-howto.md) for the full guide.

### Quick start

```
/architect-bootstrap ~/Git/my-project

Add a health check endpoint that returns pod readiness and database connectivity
```

The agent scans the workspace, builds a canvas with situational awareness, then guides you through the pipeline:

```
/create-plan        # Build atomic implementation plan + 4-layer ATDD test spec
/grill-me           # Stress-test decision tree 1 question/turn & lock choices (also post pre-flight/converge)
/pre-flight         # Audit plan confidence (soft gate ≥ 90%)
/update-plan        # Bake Path to Green into the plan (or /converge-plan to auto-loop)
/execute-plan       # Implement via Red-First ATDD closed loop
/codereview         # Multi-reviewer panel
/open-mr            # Push branch, open PR/MR, monitor CI & reviews, merge when safe
/eop                # Capture learnings into AGENTS.md, write session digest
```

### Recommended: Two-session workflow

1. **Session 1 (explore):** Run `/architect-bootstrap` to scan and understand the workspace. Brainstorm with the agent. Ask it to write a focused feature prompt.
2. **Session 2 (execute):** Open a fresh chat. Paste the feature prompt into `/architect-bootstrap`. Run the full pipeline with clean context.

Artifacts on disk bridge the gap — Session 1's canvas, `docs/brainstorms/` requirements, `.agents/plans/` artifacts, and `AGENTS.md` are read automatically by Session 2's `/architect-bootstrap`.

## Design principles

This pipeline is built on:

- **[Cynefin framework](https://en.wikipedia.org/wiki/Cynefin_framework)** — classify problems before solving them (Clear/Complicated/Complex/Chaotic)
- **[Control theory](https://en.wikipedia.org/wiki/Control_theory)** — user intent = setpoint, code = controller, tests/logs = feedback loop
- **[Hexagonal architecture](https://alistair.cockburn.us/hexagonal-architecture/)** — ports & adapters, domain independence
- **[Continuous delivery](https://continuousdelivery.com/)** — small verifiable batches, build quality in
- **[Platform Strategy](https://architectelevator.com/book/platformstrategy/)** (Gregor Hohpe) — 7 C's, Double Pyramid, abstractions not illusions
- **Probe-before-assume** — never act on assumptions when evidence is available
- **Red-First ATDD** — independent test specifier writes failing 4-layer acceptance tests first, locks the test contract, and a separate code executor turns Red into Green
- **Orchestrator mandate** — main agent coordinates and verifies; worker subagents execute all code mutations

## License

[MIT](LICENSE)
