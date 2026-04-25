# Structured Error Events

> Every subagent must report its status in a standard format. No silent failures. No ambiguous outcomes. If it ran, it reports.

**Category**: Observability | **Complexity**: Low | **Prerequisite**: [Task Envelope](../coordination/task-envelope.md)

---

## Problem

You spawn three subagents in parallel. Two return results. The third... nothing. Did it succeed? Is it still running? Did it crash? Is it hallucinating a response? You don't know, and now you can't safely proceed because the downstream task depends on all three outputs.

Silent failures are the #1 operational problem in multi-agent systems.

## Solution

Every subagent must emit a structured status event when it finishes (or when it gets stuck):

```
COMPLETED  {task_id}: completed (output: {file_path}, {size})
FAILED     {task_id}: FAILED (error: {error_message})
RUNNING    {task_id}: running (step {n}/{total}: {description})
BLOCKED    {task_id}: blocked (reason: {reason})
```

### Status Types

| Status | Meaning | Orchestrator Action |
|--------|---------|-------------------|
| `completed` | Task done, output available | Validate output against acceptance criteria |
| `FAILED` | Task failed, no usable output | Pause downstream tasks, decide retry/escalate |
| `running` | Task in progress, checkpoint available | Wait (or timeout if too long) |
| `blocked` | Task cannot proceed, needs input | Resolve blocker or escalate |

## Why This Matters

### Preventing hallucination cascades

Without structured events, this happens:
1. Agent A fails to fetch API data (gets a 401 error)
2. Agent A returns nothing, or returns an empty file
3. Agent B reads the empty file and writes an analysis anyway -- full of hallucinated data
4. The orchestrator sees a plausible-looking analysis and accepts it

With structured events:
1. Agent A emits: `FAILED api-fetch: FAILED (error: 401 Unauthorized)`
2. Orchestrator pauses Agent B (which depends on Agent A's output)
3. Orchestrator decides: fix auth and retry, or escalate

**The structured event broke the cascade at step 2** instead of letting bad data flow downstream.

## Implementation

### File-based (for AI coding agents)

Subagent writes its status as the last line of its output file:

```markdown
## Output: research-api

[... actual analysis content ...]

---
STATUS: completed | output: artifacts/research-api.md | 2.3KB
```

Or for failures:

```markdown
## Output: research-api

STATUS: FAILED | error: API endpoint returned 401, token may be expired
```

### Structured file (for code-based orchestrators)

```json
{
  "task_id": "research-api",
  "status": "FAILED",
  "error": "API endpoint returned 401",
  "error_type": "auth",
  "attempts": 1,
  "timestamp": "2026-03-15T10:30:00Z"
}
```

## Orchestrator Responsibilities

When you receive an event:

| Event | Action |
|-------|--------|
| `completed` | Check acceptance criteria. If pass → accept. If fail → retry (up to 2x). |
| `FAILED` | Log error. Pause dependent tasks. Classify error type (see [Error Handling](../../guides/error-handling.md)). Decide: retry, restart, or escalate. |
| `running` | If within timeout → wait. If past timeout → nudge or kill. |
| `blocked` | Read the reason. If you can resolve it → resolve and resume. If not → escalate. |
| No event at all | **This is the worst case.** After timeout, assume failure. Kill the agent, write a failure event yourself, proceed with error handling. |

## Completion Notification Rule

This is a **mandatory** practice, not a suggestion:

> Any agent that performs an operation must send a clear completion notification immediately upon finishing. Not embedded in a long analysis. Not after a follow-up question. Immediately.

| Format | Example |
|--------|---------|
| Success | "Task research-api completed. Output: artifacts/research-api.md" |
| Failure | "Task research-api FAILED: API returned 401, credentials may be expired" |

### Why immediate and separate?

In practice, agents tend to:
1. Complete a task
2. Start analyzing the result
3. Write a long response about the analysis
4. Mention completion somewhere in paragraph 4

The human (or orchestrator) then has to read the entire response to figure out if the task actually succeeded. A separate, immediate notification eliminates this.

---

*See also: [Error Event Template](../../templates/error-event.md), [Circuit Breaker](../resilience/circuit-breaker.md) for what happens after repeated failures.*
