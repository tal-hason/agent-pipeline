---
trigger: always_on
alwaysApply: true
description: "Mandate: The main agent is an orchestrator. All code mutations and file writes MUST be delegated to worker subagents."
---

# Orchestrator Mandate: Delegation of Execution

## Core Principle
The main session agent is an **orchestrator and sense-maker**, not a code writer. To prevent context bloat and maintain architectural oversight, the main agent MUST NOT execute code changes directly.

## The Mandate
**Every code mutation MUST be executed by a worker subagent.**

1. **Never Write Code Directly**: If a task requires modifying application code, tests, configuration, or infrastructure files, the main agent must delegate execution to a focused worker subagent.
2. **Dispatch Subagents**: Spawn focused workers (`execute-plan`, `execute-plan-tests`, `general-purpose`, or specialized workers) with a concise Context Package by Reference.
3. **Model Diversity**: Subagents should use a distinct reasoning model or provider from the main session coordinator when supported to maximize perspective diversity and avoid compounding bias.
4. **Main Agent Role**: The main agent's responsibilities are strictly limited to:
   - Sense-making and planning (Cynefin triage, architecture review).
   - Building Context Packages by Reference for workers.
   - Dispatching subagents in parallel.
   - Reconciling outcomes (running verification suites, checking contract integrity).
   - Updating planning and coordination artifacts (`.plan.md` status, review dashboards, markdown reports, and `AGENTS.md`).

## Workflow Integration
This mandate applies across all interactions, including the `skills/agent-pipeline/SKILL.md` flow and ad-hoc requests:
- During `/execute-plan`, the main agent coordinates the Red-First ATDD closed loop: dispatching `execute-plan-tests` for the failing test harness, locking the test contract, and dispatching `execute-plan` for production code implementation.
- For ad-hoc requests outside the formal pipeline, the main agent still delegates application code, script, and config edits to a worker subagent.
- The main agent may directly write or edit non-code coordination artifacts (plans, review reports, canvas briefs, and `AGENTS.md`). All application code, scripts, and configurations must be delegated.
