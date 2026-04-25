# Dead Letter Queue

> When a task fails all retries and hits the circuit breaker, don't drop it. Park it in a dead letter queue for later retry when conditions change.

**Category**: Resilience | **Complexity**: Low | **Prerequisite**: [Circuit Breaker](circuit-breaker.md)

---

## Problem

A task fails. You retry 3 times. It still fails. The circuit breaker opens. The orchestrator moves on. But the task is now... gone. Nobody remembers it existed. When the API comes back online 2 hours later, the task is forgotten.

In a system where agents are stateless and sessions are ephemeral, untracked failed tasks simply vanish.

## Solution

When a task exhausts all retries, write it to a dead letter file instead of discarding it. Periodically sweep the dead letter queue and retry tasks whose failure conditions may have resolved.

### Dead Letter Format

```json
{
  "task_id": "fetch-market-data-2026-03-15",
  "original_task": "Fetch BTC/USD market data from the exchange API",
  "error": "429 Too Many Requests",
  "error_type": "transient",
  "attempts": 3,
  "first_attempt": "2026-03-15T09:00:00Z",
  "last_attempt": "2026-03-15T09:15:00Z",
  "context_summary": "Part of daily market analysis pipeline. Depends on this data for the morning report.",
  "retry_after": "2026-03-15T10:00:00Z",
  "priority": "medium"
}
```

### Directory Structure

```
dead-letters/
  fetch-market-data-2026-03-15.json
  deploy-staging-2026-03-14.json
  archive/
    old-letters-2026-02.json    # Archived after 7 days
```

## Lifecycle

```
Task fails all retries
  │
  ▼
Dead Letter created
  │
  ├─── Periodic sweep: conditions resolved? ──→ Retry ──→ Success ──→ Delete
  │
  ├─── Periodic sweep: still failing? ──→ Keep in queue
  │
  ├─── 7 days old ──→ Archive
  │
  └─── Human reviews ──→ Manual retry, cancel, or fix root cause
```

## Sweep Strategy

| Interval | Action |
|----------|--------|
| Every hour | Check dead letters where `retry_after` has passed. Retry transient errors. |
| Daily | Review all dead letters. Archive anything > 7 days old. |
| On provider recovery | When a circuit breaker transitions from OPEN → CLOSED, check for related dead letters. |

### Retry Logic During Sweep

```
For each dead letter:
  1. Is the error_type transient?
     No → Skip (needs human intervention)
  2. Has retry_after passed?
     No → Skip (too soon)
  3. Is the related circuit breaker CLOSED?
     No → Skip (provider still down)
  4. Retry the task
     Success → Delete the dead letter
     Failure → Update attempts count, set new retry_after, keep in queue
```

## When to Use Dead Letters

| Scenario | Use Dead Letter? | Why |
|----------|:---:|-----|
| API rate limit (429) | Yes | Will resolve when rate limit resets |
| Service outage (503) | Yes | Will resolve when service recovers |
| Auth failure (401) | Yes | Will resolve when credentials are refreshed |
| Bad request (400) | **No** | Your request is wrong, retrying won't help |
| Logic error (wrong output) | **No** | The task spec needs fixing, not retrying |
| Budget exceeded | **Maybe** | If budget will be replenished, yes |

## Priority Levels

Not all dead letters are equally important:

| Priority | Meaning | Sweep Behavior |
|----------|---------|---------------|
| `high` | Blocking other work | Retry every 15 minutes |
| `medium` | Important but not blocking | Retry every hour |
| `low` | Nice to have | Retry once daily |

## Anti-Patterns

### Infinite queue growth

Dead letters accumulate forever because nobody reviews them.

**Fix**: Enforce the 7-day archive rule. Anything older than 7 days is either no longer relevant or needs human intervention -- not more automated retries.

### Retrying logic errors

A task that produced wrong output gets dead-lettered and retried with the same bad prompt. It produces the same wrong output.

**Fix**: Only dead-letter transient failures (network, rate limits, outages). Logic failures need a different fix -- usually rewriting the task envelope.

### No context in the dead letter

The dead letter says "task failed" but doesn't explain what the task was for or what depends on it.

**Fix**: Include `context_summary` -- one sentence explaining why this task matters and what's waiting for it.

---

*See also: [Dead Letter Template](../../templates/dead-letter.json), [Circuit Breaker](circuit-breaker.md) for the pattern that feeds into dead letters.*
