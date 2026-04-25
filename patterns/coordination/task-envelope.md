# Task Envelope

> Every subagent task is wrapped in a structured envelope: goal, acceptance criteria, inputs, allowed tools, budget, and stop conditions. No more "just figure it out" prompts.

**Category**: Task Decomposition | **Complexity**: Low | **Prerequisite**: [File Blackboard](file-blackboard.md)

---

## Problem

You spawn a subagent with a vague instruction like "analyze this data." The subagent:
- Doesn't know what "good enough" looks like
- Reads irrelevant files, burning tokens
- Runs 30 tool calls when 5 would suffice
- Fails silently instead of reporting the problem
- Produces output that doesn't match what you needed

## Solution

Package every task as a **Task Envelope** -- a structured brief that tells the subagent exactly what to do, how to know it's done, and when to stop trying.

### Full Envelope

```markdown
## Task: {task-id}
- **Goal**: One sentence describing the desired outcome
- **Acceptance Criteria**: Objective conditions for "done" (test passes, file exists, format matches)
- **Inputs**: File paths + one-line summaries (not full contents)
- **Allowed Tools**: Explicit list of permitted operations
- **Risk Level**: low / medium / high
- **Budget**: Max iterations, max tool calls, or deadline
- **Checkpoint**: When to save intermediate progress
- **Stop Conditions**: When to give up and report back
```

### Lightweight Envelope

For simple tasks (< 5 tool calls expected):

```markdown
## Task: {task-id}
- **Goal**: One sentence
- **Inputs**: File paths
- **Output**: Expected file path and format
- **Stop if**: Condition to bail out
```

## Field Guide

| Field | Purpose | Bad Example | Good Example |
|-------|---------|-------------|--------------|
| Goal | What to produce | "Look into the API" | "Write a comparison table of REST vs GraphQL for our use case" |
| Acceptance Criteria | How to verify done | (none) | "Output has >= 3 rows, includes latency and cost columns" |
| Inputs | What to read | "Check the project files" | "`data/api-bench.json` (latency benchmarks, 50 entries)" |
| Allowed Tools | Scope guard | (none) | "Read files, run tests. No web requests, no file deletion." |
| Budget | Cost ceiling | (none) | "Max 10 tool calls" |
| Stop Conditions | When to bail | (none) | "If benchmark file is missing, report and stop" |

## Decomposition Principles

### 1. Each task must be independently retryable

If a task fails, you should be able to re-run it without affecting other tasks. This means:
- No shared mutable state between tasks
- Each task reads its own inputs and writes its own outputs
- Dependencies are serial (task B waits for task A), not entangled

### 2. Estimate tool calls -- split if > 10

A task that needs 10+ tool calls is likely doing two things. Split it. Smaller tasks are cheaper to retry and easier to validate.

### 3. Dependencies = serial. No dependencies = parallel.

```
Task A ──→ Task B ──→ Task C     (serial: B needs A's output)
Task D ─┐
Task E ─┼──→ Orchestrator merge  (parallel: D, E, F are independent)
Task F ─┘
```

### 4. No recursive spawning

Subagents do not spawn other subagents. Ever. The orchestrator is the only dispatcher. This prevents:
- Uncontrolled cost growth
- Circular dependencies
- Loss of oversight

## Dispatch Best Practices

### Give context, not just the query

Bad:
> "What's the best database for this project?"

Good:
> "What's the best database for this project? Context: we need < 10ms reads, expect 100K rows, team has PostgreSQL experience, budget is $0/month (self-hosted). Compare PostgreSQL, SQLite, and DuckDB."

Subagents don't have the orchestrator's global context. Tell them **why** you're asking and **what constraints** matter.

### Evaluate before accepting

After a subagent returns, check the output against acceptance criteria before using it:

```
Subagent returns →
  Meets acceptance criteria?
    Yes → Accept, write to global state
    No →
      First retry? → Give specific feedback, retry
      Second retry? → Give specific feedback, retry
      Third time? → Accept partial result or escalate
```

### Give specific feedback on retry

Bad:
> "Try again."

Good:
> "The comparison table is missing the cost column. Add estimated monthly cost for each option at 100K rows."

Tell the subagent **what's missing**, not just that it's wrong.

## Cost Impact

Task envelopes reduce cost in three ways:

| Mechanism | Saving |
|-----------|--------|
| Scoped inputs (paths, not full files) | 40-60% fewer input tokens |
| Budget limits (max tool calls) | Prevents runaway exploration |
| Acceptance criteria (clear "done") | Fewer retry rounds |

A well-written envelope typically saves 2-3x the tokens of an unstructured prompt, because the subagent doesn't waste cycles figuring out what you want.

## Advanced: Structured Handoff Schema

When the task is complex, a normal envelope is still not enough. Add a **Structured Handoff Schema** on top of it:

```markdown
- **Objective**: What must be achieved
- **Context**: Constraints the agent cannot infer on its own
- **Acceptance Criteria**: How "done" will be validated
- **Interface Contract**: What inputs, outputs, or APIs must not drift
- **Anti-patterns**: Failure modes to actively avoid
- **Review Checklist**: What the reviewer must verify
```

This is especially useful when the task will pass through multiple agents. Research on MetaGPT suggests the real gain came from SOP-style structured outputs, not from giving agents fancy role personas. The format constrained what downstream agents could misread.

### For algorithm work

Add a seventh field: **Data Integrity Requirements**

```markdown
- **Data Integrity Requirements**:
  - No look-ahead leakage
  - Point-in-time data only
  - Purge + embargo splits where overlap can leak signal
  - Realistic transaction costs and slippage
```

This field exists because algorithm agents will happily produce a beautiful false edge if the handoff does not explicitly forbid it.

---

*See also: [Templates](../../templates/task-envelope.md) for copy-paste formats, [Context Management](../../guides/context-management.md) for input optimization.*
