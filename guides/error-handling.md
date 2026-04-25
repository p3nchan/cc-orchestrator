# Error Handling Guide

> Classify every error. Retry transients. Restart from checkpoint on logic failures. Escalate when the circuit breaker opens. Never retry blindly.

---

## Error Classification

| Type | Examples | Correct Response | Wrong Response |
|------|---------|-----------------|----------------|
| **Transient** | 429 rate limit, 503 timeout, network blip | Retry with exponential backoff (max 3) | Retry infinitely |
| **Auth** | 401 unauthorized, 403 forbidden, expired token | Check credentials, refresh token, or escalate | Retry (will always fail) |
| **Logic** | Output doesn't meet acceptance criteria, wrong format | Restart task with better context | Retry with same prompt |
| **State** | Corrupted checkpoint, incomplete file write | Rollback to last good checkpoint | Continue with corrupted state |
| **Catastrophic** | 5+ consecutive failures, budget exceeded | Circuit breaker + escalate to human | Keep retrying |

## The 5-Layer Fallback Pyramid

When a task fails, work down the pyramid. Each layer is more aggressive (and more expensive) than the last.

```
Layer 1: Normal Execution
  │  ← Task runs with full context
  │  Failure?
  ▼
Layer 2: Auto-Repair
  │  ← Clean encoding issues, truncate malformed tool output, fix obvious errors
  │  Still failing?
  ▼
Layer 3: Context Reduction
  │  ← Keep only system prompt + last 10 conversation turns
  │  Still failing?
  ▼
Layer 4: Minimal Context
  │  ← Only the most recent instruction, no history
  │  Still failing?
  ▼
Layer 5: Fresh Restart
  │  ← New session, load checkpoint summary only
  │  Still failing?
  ▼
Escalate to human
```

Most failures resolve at Layer 1 (retry) or Layer 2 (auto-repair). If you're hitting Layer 4+, something is fundamentally wrong.

## Retry vs Restart vs Escalate Decision Tree

```
Error occurs →
  │
  ├─ Is it transient? (429, 503, network timeout)
  │   ├─ Yes → Retry (max 3, exponential backoff)
  │   │         └─ Still failing after 3 retries? → Circuit breaker → Escalate
  │   │
  │   └─ No → Continue below
  │
  ├─ Has the task made progress since the last checkpoint?
  │   ├─ Yes → Restart from checkpoint (once)
  │   │         └─ Fails again? → Escalate
  │   │
  │   └─ No → Continue below
  │
  ├─ Has the budget cap or circuit breaker been hit?
  │   ├─ Yes → Escalate to human immediately
  │   │
  │   └─ No → Restart with fresh context (once)
  │           └─ Fails again? → Escalate
  │
  └─ (Should never reach here. If it does → Escalate)
```

## Fix Loop Hard Cap

Fix loops need a ceiling. Research across agent failure studies keeps finding the same pattern: the first few rounds help, then the loop starts reinforcing the wrong approach.

Use two caps:

| Cap | Rule | Why |
|-----|------|-----|
| **Soft cap: 3 rounds** | Expect real progress in the first 2-3 passes | Research summarized in the MAST taxonomy found overthinking loops made up 39.3% of errors in a financial processing benchmark |
| **Hard cap: 10 rounds** | Stop and escalate with the current best state plus an unresolved-issues list | Past this point, you are paying to churn |

Failure studies on SWE-bench Verified also found failed issues consumed 3.5x more interaction steps than successful ones. Long fix loops are usually a warning, not a sign of persistence paying off.

## Expert-Executor Escalation

When an executor stalls at round 2, stop asking it to "try a different fix" blindly. Escalate to the orchestrator or a senior agent and ask a higher-order question:

- Is the problem in the spec or the implementation?
- Are we validating the right thing?
- Do we need a different approach entirely?

This is the **Expert-Executor** pattern. Research on failed issue recovery found it recovered 22.2% of previously failed issues, versus 7% for single-agent retry.

## Retry Best Practices

### Exponential backoff with jitter

```
Attempt 1: Wait 1s + random(0-500ms)
Attempt 2: Wait 2s + random(0-500ms)
Attempt 3: Wait 4s + random(0-500ms)
(Give up after 3)
```

The jitter prevents synchronized retries when multiple agents hit the same rate limit simultaneously.

### What to retry

| Error | Retry? | Max Attempts | Backoff |
|-------|:---:|:---:|---------|
| 429 Rate Limit | Yes | 3 | Exponential |
| 503 Service Unavailable | Yes | 3 | Exponential |
| Network timeout | Yes | 2 | Linear (30s) |
| Connection reset | Yes | 2 | Linear (10s) |

### What NOT to retry

| Error | Why |
|-------|-----|
| 401/403 (auth) | Credentials are wrong. Fix them first. |
| 400 (bad request) | Your request is malformed. Fix the request. |
| Output quality (hallucination, wrong format) | The model understood you; it just got it wrong. Retrying the same prompt rarely helps. |
| Budget exceeded | More retries = more cost = more over budget. |

## Restart Best Practices

Restarting means: terminate the current attempt, load the checkpoint, and spawn a fresh agent with clean context.

### When restart beats retry

- The agent's context window is polluted with error messages from previous attempts
- The agent is looping (trying the same approach repeatedly)
- The error is a logic error (output is wrong, not missing)

### Restart checklist

1. Write a checkpoint of current state (even if it's a failure checkpoint)
2. Note what was tried and why it failed (include this in the new agent's context)
3. Spawn a fresh agent with:
   - The checkpoint summary
   - What was tried
   - What to do differently
4. Give the fresh agent a budget that accounts for the spent budget

## Lessons From Production

### Silent proxy failures

An API proxy between your agents and the model provider crashes. Every request returns a generic error. Agents retry endlessly because each error looks like a transient failure.

**Pattern**: 5+ identical consecutive errors = treat as circuit breaker trigger, regardless of error code.

### Failover avalanche

Primary provider goes down. All agents fail over to the secondary provider at the same moment. Secondary gets overwhelmed and rate-limits. Now both providers are down.

**Fix**: Add random jitter (0-30s) before failover. Use per-provider circuit breakers. Don't send all traffic to one backup.

### The timeout that eats queued messages

A request times out after 90 seconds. Three more requests queued up behind it. Timeout resolves, but now four responses arrive at once.

**Fix**: After a timeout, resume one request at a time. Don't blast the queue.

### Retrying logic errors

Agent produces a wrong analysis. You retry. Same wrong analysis. Retry. Same. Three retries wasted.

**Fix**: Logic errors need a different prompt or different context, not the same prompt repeated. Give specific feedback about what's wrong.

### Validation Retreat

The agent cannot fix the code cleanly, so it weakens the tests instead. Now the suite is green, but the bug remains.

**Pattern**: Tests changed during the fix loop, but the change is not justified by the spec.

**Fix**: Every review pass should check whether test edits happened, whether they were necessary, and whether the system became more correct or just easier to satisfy.

## Error Event Format

Every error should be logged in a consistent format:

```
{task_id}: FAILED
  Type: transient | auth | logic | state | catastrophic
  Error: {error message}
  Attempt: {n} of {max}
  Action: retry | restart | escalate
  Next: {what happens now}
```

This makes it possible to grep your logs for patterns and build pre-flight checks for recurring errors.

## Building a Debug Knowledge Base

Over time, you'll encounter the same errors repeatedly. Track them:

```markdown
### [Date] Error Name

**Symptom**: What you observed
**Root Cause**: What actually went wrong
**Fix**: What resolved it
**Prevention**: How to avoid it in the future
```

When a new session starts and hits a familiar-looking error, search the debug KB first. Don't re-diagnose known issues.

### Promotion rule

If the same error appears 3+ times in 14 days, promote it from the debug KB to a **pre-flight check** -- something the orchestrator verifies before starting any task.

---

*See also: [Circuit Breaker](../patterns/resilience/circuit-breaker.md), [HITL Escalation](../patterns/communication/hitl-escalation.md), [Structured Error Events](../patterns/communication/structured-error-events.md)*
