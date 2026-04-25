# Circuit Breaker

> Track failures per tool or API provider. Three consecutive failures = stop calling. Wait, then probe once. This prevents the most expensive failure mode in multi-agent systems: cascading retry storms.

**Category**: Error Handling | **Complexity**: Medium | **Prerequisite**: None

---

## Problem

Your agent hits a rate limit (429). It retries. Retries again. Meanwhile, a second agent is also retrying. Now both are hammering the API, making the rate limit worse. Your third agent fails over to an alternative provider -- which also gets rate-limited because everyone just failed over at the same time.

This is a **retry storm**, and it's the fastest way to burn through your API budget doing nothing useful.

## Solution

Implement a circuit breaker with three states:

```
CLOSED (normal operation)
  │
  │  3 consecutive failures within 60s
  ▼
OPEN (all calls blocked)
  │
  │  Wait 5 minutes
  ▼
HALF-OPEN (try one call)
  │
  ├── Success → CLOSED (resume normal)
  │
  └── Failure → OPEN (extend cooldown)
```

### State Machine

| State | Behavior | Transition |
|-------|----------|------------|
| **CLOSED** | Calls proceed normally. Failures are counted. | 3 consecutive failures in 60s → OPEN |
| **OPEN** | All calls immediately fail without executing. | After 5 min cooldown → HALF-OPEN |
| **HALF-OPEN** | Allow exactly one probe call. | Success → CLOSED. Failure → OPEN (double cooldown). |

### Configuration

| Parameter | Default | Adjust When |
|-----------|---------|-------------|
| Failure threshold | 3 | Lower for expensive APIs, higher for cheap ones |
| Time window | 60 seconds | Shorter for real-time systems |
| Cooldown period | 5 minutes | Longer for APIs with known long outages |
| Cooldown multiplier | 2x on repeated OPEN | Cap at 30 minutes |

## Implementation

### For AI agents (no code needed)

You don't need a library. Track state in a file:

```json
{
  "provider": "openai",
  "state": "OPEN",
  "consecutive_failures": 3,
  "last_failure": "2026-03-15T10:30:00Z",
  "cooldown_until": "2026-03-15T10:35:00Z",
  "cooldown_minutes": 5
}
```

The orchestrator checks this file before dispatching any task that uses the provider. If state is OPEN and current time < cooldown_until, don't dispatch.

### For code-based orchestrators

```python
class CircuitBreaker:
    def __init__(self, threshold=3, window_sec=60, cooldown_sec=300):
        self.threshold = threshold
        self.window = window_sec
        self.cooldown = cooldown_sec
        self.failures = []
        self.state = "CLOSED"
        self.open_until = None

    def record_failure(self):
        now = time.time()
        self.failures = [t for t in self.failures if now - t < self.window]
        self.failures.append(now)
        if len(self.failures) >= self.threshold:
            self.state = "OPEN"
            self.open_until = now + self.cooldown

    def record_success(self):
        self.failures = []
        self.state = "CLOSED"

    def can_call(self):
        if self.state == "CLOSED":
            return True
        if self.state == "OPEN" and time.time() >= self.open_until:
            self.state = "HALF_OPEN"
            return True  # Allow one probe
        return False
```

## Multi-Provider Failover

Circuit breakers become critical when you have multiple providers:

```
Primary (Claude) ──→ [OPEN] ──→ Fallback (GPT) ──→ [OPEN] ──→ Stop
```

### The Failover Avalanche Problem

When your primary provider goes down:
1. All agents fail over to the secondary
2. Secondary gets hit with 3x normal load
3. Secondary rate-limits
4. Both providers are now in OPEN state
5. Nothing works

### Prevention

| Strategy | How |
|----------|-----|
| **Staggered failover** | Don't fail over all agents at once. Add random jitter (0-30s) before failover |
| **Per-provider breakers** | Each provider has its own independent circuit breaker |
| **Graceful degradation** | If all providers are OPEN, queue tasks instead of dropping them |
| **Capacity-aware routing** | If provider B is already at 80% capacity, don't route failover traffic to it |

## What Triggers the Breaker

| Error Type | Should Trigger? | Why |
|------------|:-:|-----|
| 429 (rate limit) | Yes | Provider is overloaded, more calls make it worse |
| 503 (service unavailable) | Yes | Provider is down |
| Network timeout | Yes | Connection problems won't fix in the next second |
| 401 (auth failure) | **No** | This is a config issue. Fix the token, don't wait. |
| 400 (bad request) | **No** | This is your bug. Retrying won't help. |
| Logic error (bad output) | **No** | The API worked fine; your prompt was wrong. Use [Task Envelope](../coordination/task-envelope.md) retry logic instead. |

## Real Failure Modes

### The silent proxy crash

An API proxy between you and the provider crashes. Every request gets a generic error. Your agents retry endlessly because the error looks transient.

**Fix**: If you get 5+ identical errors in a row, treat it as a circuit breaker trigger regardless of the error code. Consecutive identical errors = systemic failure.

### The timeout that eats your queue

A request times out after 90 seconds. During that time, 3 more requests queue up behind it. The timeout resolves, but now all 4 responses arrive at once and overwhelm your agent's context.

**Fix**: After a timeout, don't immediately retry AND send queued requests. Resume one at a time.

### The recovery spike

Provider comes back online. All your HALF-OPEN breakers probe at the same second. Provider gets hammered again.

**Fix**: Add jitter to HALF-OPEN probes. Don't all probe at exactly cooldown_until.

---

*See also: [Error Handling Guide](../../guides/error-handling.md) for the full retry-vs-restart decision tree.*
