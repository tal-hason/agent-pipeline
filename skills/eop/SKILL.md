---
name: eop
description: >-
  Stage 10 (End of Pipeline) of the agent pipeline. Closes the pipeline session
  by updating workspace documentation, checking PR/issue status, capturing continual
  learnings into AGENTS.md, and compressing context into a session digest.
---

# End of Pipeline

**Invoke via:** `/eop`

**You are the coordinator.** Execute Steps 1–5 in order. This is the **final stage** of the agent pipeline — it updates docs, checks issue/PR tracking, captures session learnings into `AGENTS.md`, and compresses context for future sessions.

**Requires:** `/open-mr` merged (or `/codereview` complete if shipping was skipped / user closing early).

**Pipeline:** See [Agent Pipeline Contract](../agent-pipeline/SKILL.md). This skill is terminal — no further stages.

## Execution checklist

| Step | Who | Action | Gate |
| :--- | :--- | :--- | :--- |
| 1 | Main agent | **Context recall** (`AGENTS.md`, session artifacts) | Prior context loaded |
| 2 | Main agent | **Update project/folder docs** | Docs current (or no-op confirmed) |
| 3 | Main agent | **Issue & PR/MR status check** | Linked issues/PRs updated (or skipped) |
| 4 | Main agent | **Continual Learning extraction** | `AGENTS.md` updated (or no-op confirmed) |
| 5 | Main agent | **Context compression** | Session digest written on disk |

---

## Step 1: Context Recall

Read workspace and user-level `AGENTS.md` plus the session's produced artifacts (plan, pre-flight, execution outcome, code review, open-mr outcome).

---

## Step 2: Update Project/Folder Docs

Review and update documentation artifacts affected by the session's work. Follow the documentation-first principle — docs should reflect the current state before the session closes.

### What to check
1. **Task / repo `README.md`** — update with new features, changed APIs, env vars, or merged PR/MR links.
2. **`CHANGELOG` or release notes** — update if the project maintains one.
3. **Architecture / coordination docs** — update only with work actually executed (never speculative planned work).

If the session was purely exploratory (bootstrap only, no execution): skip this step.

---

## Step 3: Issue & PR/MR Status Update (Optional)

If the work is linked to a GitHub Issue, GitLab Issue, Linear ticket, or PR/MR:
1. Confirm with the user before posting external comments or changing issue states.
2. Add a concise completion comment summarizing the PR/MR URL, commit SHAs, and verification results.
3. Update or close the issue only upon explicit user confirmation (or record *"Skipped"* if no tracker is linked).

---

## Step 4: Continual Learning Extraction

Mine the **current session** for durable, cross-session learnings. Update the appropriate `AGENTS.md` file(s).

### What qualifies as a learning

| Category | Example |
| :--- | :--- |
| **User preferences** | Commit style, review gate behavior, naming conventions |
| **Workspace facts** | Architecture boundaries, repo conventions, hardware constraints |
| **Workflow patterns** | "Always do X before Y", "Skip Z when condition" |
| **Tool/API discoveries** | Environment quirks, rate limits, schema details |
| **Anti-patterns observed** | Approaches that failed and why |

### What does NOT qualify

- Transient state (today's date, current branch, one-off flags)
- Task-specific details already captured in plan/review artifacts
- Opinions not validated by evidence in the session

### Extraction procedure
1. **Scan the session** for decisions, corrections, surprises, and repeated patterns.
2. **Draft delta** — new bullet points or updates to existing ones, grouped by section.
3. **Deduplicate** — check existing `AGENTS.md` content; skip if already present.
4. **Apply** — append or update workspace `AGENTS.md` (and user-level `AGENTS.md` for cross-workspace learnings).
5. **Report** — show the user what was added/changed (diff-style).

If the session produced no durable learnings: state *"No new learnings to extract."* and proceed to Step 5.

---

## Step 5: Context Compression

Produce a **session digest** — a compressed summary that bridges this session to the next.

```markdown
## Session Digest: {date} — {one-line intent}

### Pipeline stages completed
- /architect-bootstrap: {outcome}
- /create-plan: {plan slug}
- /grill-me: {decisions locked or skipped}
- /pre-flight: {confidence%, run count}
- /update-plan / /converge-plan: {rounds / summary or skipped}
- /execute-plan: {outcome}
- /codereview: {verdict}
- /open-mr: {url + merged|paused|skipped}

### Key decisions made
- {decision 1}
- {decision 2}

### Artifacts produced
| Artifact | Path |
| :--- | :--- |
| Plan | `{path}` |
| Pre-flight | `{path}` |
| Code review | `{path}` |
| Open-MR outcome | `{path}` |
| Canvas | `{path}` |

### Docs updated
- {doc path}: {what changed}

### Issue / PR tracking
- {issue or PR reference}: {action taken or "Skipped"}

### Open threads (carry forward)
- {anything unresolved, deferred, or flagged for next session}
```

Write to: `.agents/digests/{slug}-session-digest_{YYYY-MM-DD_HHMM}.md` (or `.cursor/plans/{slug}-session-digest_{YYYY-MM-DD_HHMM}.md`).

If only a partial pipeline was run, include only applicable sections.

---

## Closing CTA

End with:

> *"Pipeline complete. Session learnings captured in `AGENTS.md`. Digest at `{path}`."*

---

## Anti-Patterns

- Storing transient/ephemeral state as "learnings" (branch names, one-off dates)
- Skipping context compression when the session was productive
- Writing duplicate entries already in `AGENTS.md`
- Overwriting existing learnings instead of appending/updating
- Posting external tracker comments or transitioning issues without user approval
- Updating docs with speculative/planned work that wasn't actually executed

## Pipeline position

| # | Skill | Actor |
| :--- | :--- | :--- |
| 1 | `/architect-bootstrap` | main agent (+ scan panel) |
| 2 | `/create-plan` | main agent (+ evidence explorers) |
| 3 | `/grill-me` | main agent (interactive decision-tree interview) |
| 4 | `/pre-flight` | main agent (+ `pre-flight` auditors) |
| 5 | `/update-plan` | main agent (+ conditional explorers) |
| 6 | `/converge-plan` | main agent (orchestrates pre-flight ⇄ update-plan) |
| 7 | `/execute-plan` | main agent (+ test specifier + executor) |
| 8 | `/codereview` | main agent (+ specialized review panel) |
| 9 | `/open-mr` | main agent (+ CI & review monitor) |
| 10 | `/eop` | **main agent** (terminal — no workers) |
