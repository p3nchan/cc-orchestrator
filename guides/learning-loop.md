# Learning Loop Guide

> Every failure is data. Track it, analyze it, prevent it from recurring. Systems that don't learn from failures repeat them.

---

## The Error Learning SOP

When any task fails, follow this sequence:

```
1. Record the symptom and error signature
2. Analyze the root cause (not just the surface error)
3. Document the fix
4. Write prevention steps
5. Add to the Debug Knowledge Base
6. If the same error pattern appeared 3+ times in 14 days → create a pre-flight check
```

## Debug Knowledge Base

Maintain a running log of errors, their causes, and their fixes. This is the single most valuable artifact in a long-running multi-agent system.

### Format

```markdown
### [YYYY-MM-DD] Descriptive Error Name

**Symptom**: What you observed (error message, unexpected behavior)
**Root Cause**: The actual underlying problem
**Fix**: What resolved it immediately
**Prevention**: How to avoid it in the future
**Status**: Fixed / Monitoring / Recurring
```

### Example entries

```markdown
### [2026-03-10] Silent API Proxy Crash

**Symptom**: All API calls returning generic "connection refused" error.
Agents retrying endlessly, burning budget.
**Root Cause**: The API proxy process crashed and wasn't restarted.
No health check was monitoring the proxy.
**Fix**: Restarted the proxy process manually.
**Prevention**: Added hourly health check that pings the proxy endpoint.
Circuit breaker now triggers on 5+ identical consecutive errors.
**Status**: Fixed

### [2026-03-12] Failover Avalanche

**Symptom**: Primary API goes down. All agents failover to secondary
simultaneously. Secondary rate-limits. Both providers now in OPEN state.
**Root Cause**: No jitter in failover logic. All agents switch at the
exact same moment.
**Fix**: Added random 0-30s jitter before failover.
**Prevention**: Per-provider circuit breakers. Staggered failover.
Never send 100% of traffic to a single backup.
**Status**: Fixed
```

## Pre-Flight Checks

When an error pattern becomes recurring (3+ times in 14 days), promote it from a debug KB entry to an automated pre-flight check.

### What is a pre-flight check?

A verification step that runs **before** task execution, catching known failure conditions early:

```markdown
## Pre-Flight Checks

Before starting any API-dependent task:
- [ ] API proxy is responding (ping endpoint)
- [ ] Auth credentials are valid (test call)
- [ ] Rate limit status is below 80% (check headers from last call)
- [ ] Circuit breaker states are all CLOSED
```

### Graduated promotion

| Stage | When | What |
|-------|------|------|
| Debug KB entry | First occurrence | Document the error |
| Watch list | Second occurrence | Flag it for attention |
| Pre-flight check | Third occurrence | Automate the prevention |
| Architecture change | Persistent despite pre-flight | Redesign the system to eliminate the failure mode |

## Weekly Retrospective

Run a brief retrospective weekly (or after completing a major task). Cover:

| Question | Purpose |
|----------|---------|
| What tasks succeeded this week? | Identify what's working |
| What tasks failed? | Identify patterns |
| What were the top 3 failure causes? | Focus improvement effort |
| How many escalations happened? | Calibrate autonomy level |
| What was the cost breakdown by project/model? | Identify waste |
| What system/config changes are needed? | Action items |

### Output format

```markdown
## Weekly Retro — [Date Range]

### Completed
- [list]

### Failed
- [list with root causes]

### Top Failure Causes
1. [cause] — [count] occurrences — [action]
2. [cause] — [count] occurrences — [action]
3. [cause] — [count] occurrences — [action]

### Metrics
| Metric | This Week | Target |
|--------|-----------|--------|
| Task success rate | 87% | > 85% |
| Escalation rate | 12% | 10-15% |
| Avg retries per task | 1.3 | < 2 |
| Cost per successful task | $0.45 | Improving |

### Action Items
- [ ] [specific action]
- [ ] [specific action]
```

## Quarterly Harness Audit

Every quarter, review every pattern and rule in your orchestration setup. For each component, ask:

```
1. What assumption about model capability does this component encode?
2. Is that assumption still true with current models?
3. What's the worst case if we remove this component? Is it reversible?
```

### Why this matters

Models improve. A guardrail you added because GPT-4 couldn't handle X might be unnecessary now that GPT-5 can. Conversely, new capabilities might enable new patterns you haven't considered.

### Rules

- Only remove one component at a time
- Observe for at least one week before removing another
- If removing a component causes a failure, add it back immediately
- Document every removal decision and its outcome

## Key Metrics to Track

| Metric | Target | What It Tells You |
|--------|--------|-------------------|
| Task success rate | > 85% | Overall system health |
| Escalation rate | 10-15% | Autonomy calibration |
| Average retries per task | < 2 | Error handling efficiency |
| Cost per successful task | Declining trend | System optimization over time |
| Stall rate (tasks stuck > 4h) | < 5% | Deadlock and timeout detection |
| Time to recovery (after failure) | Declining trend | Resilience improvement |

Track these weekly. Trends matter more than absolute numbers. A rising retry rate means something is degrading. A declining cost per task means your optimizations are working.

---

*See also: [Error Handling Guide](error-handling.md) for classification and response, [HITL Escalation](../patterns/communication/hitl-escalation.md) for calibrating the escalation rate.*
