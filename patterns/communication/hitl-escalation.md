# HITL Escalation

> Human-in-the-loop is not a failure mode -- it's a feature. Classify every action into three tiers: autonomous, notify, or gate. Target 10-15% escalation rate.

**Category**: Safety & Control | **Complexity**: Low | **Prerequisite**: None

---

## Problem

Fully autonomous agents make irreversible mistakes. Fully manual systems waste the human's time on trivial decisions. You need a middle ground that lets agents run fast on safe operations while stopping for the dangerous ones.

## Solution

Classify every agent action into three tiers:

| Tier | Risk Profile | Agent Behavior |
|------|-------------|---------------|
| **Autonomous** | Low risk, reversible | Execute immediately. No human involvement. |
| **Notify** | Medium risk, or important milestone | Execute, then inform the human after the fact. |
| **Gate** | High risk, irreversible, or costly | Stop. Present options. Wait for human approval. |

### Examples

| Action | Tier | Why |
|--------|------|-----|
| Read files, search code | Autonomous | Zero risk, zero side effects |
| Run tests | Autonomous | Reversible, contained |
| Write files in workspace | Autonomous | Reversible (git), contained |
| Phase completion | Notify | Human should know progress |
| Cost threshold exceeded | Notify | Transparency on spend |
| Deploy to production | Gate | Irreversible side effects |
| Send external communications | Gate | Reputation risk |
| Delete data | Gate | Irreversible |
| Change system config | Gate | Can break other agents |
| Exceed budget by > 50% | Gate | Financial risk |

## Decision Card Format

When an agent hits a Gate tier action, it should present a **Decision Card** -- not a vague "what should I do?" but a structured recommendation:

```markdown
## Decision Required -- [Project] / [Task]

**Context**: [1-2 sentences of background]

**Options**:
  A. [Option] -- [Pros / Cons]
  B. [Option] -- [Pros / Cons]
  C. [Option] -- [Pros / Cons]

**Recommended**: A -- [One sentence justification]
**Risk**: [What could go wrong with the recommendation]
**Default**: If no response in 24h -> [Default action, usually the safest option]
```

### Good vs Bad Escalation

Bad:
> "The API is returning errors. What should I do?"

Good:
> "The payment API has returned 503 errors for the past 10 minutes.
> A. Wait 30 min and retry (low risk, might resolve on its own)
> B. Switch to backup provider (medium risk, untested in this workflow)
> C. Skip this task and continue with others (low risk, delays the payment feature)
> Recommended: A -- this provider has had similar outages that resolve within 20 min.
> Default: If no response in 1h, proceed with A."

The human should be able to reply with just a letter. Minimize their cognitive load.

## Escalation Triggers

| Trigger | Tier |
|---------|------|
| Same task failed >= 3 retries | Gate |
| Single task cost > 150% of budget | Gate |
| Unknown error pattern (never seen before) | Gate |
| Domain knowledge needed (business logic, strategy) | Gate |
| Any external action (email, API submission, deploy) | Gate |
| Phase or milestone completed | Notify |
| Cost approaching budget limit | Notify |

## Calibrating Your Escalation Rate

| Rate | Diagnosis |
|------|-----------|
| < 5% | Agent is too autonomous. It's probably making risky decisions silently. |
| 5-10% | Slightly aggressive. Fine if your agent handles edge cases well. |
| **10-15%** | **Sweet spot.** Human stays informed without being bottlenecked. |
| 15-25% | Agent is too cautious. Review tier classifications -- some Gates are probably Autonomous. |
| > 25% | The human is doing the work. The agent is just a messenger. |

Track your escalation rate weekly. If it drifts, adjust tier classifications.

## Anti-Patterns

### Over-escalation

The agent asks for permission on every step. The human develops **approval fatigue** and starts rubber-stamping everything -- defeating the purpose of the gate.

**Fix**: Be aggressive about classifying low-risk, reversible actions as Autonomous. The human doesn't need to approve file reads.

### Under-escalation

The agent auto-deploys a broken build, sends a malformed email, or burns $50 on retry loops -- all without asking.

**Fix**: When in doubt, escalate. The cost of a 5-minute wait for human approval is always less than the cost of an irreversible mistake.

### Vague requests

The agent escalates with "I'm stuck, what should I do?" -- forcing the human to diagnose the problem and design the solution.

**Fix**: Require the Decision Card format. The agent must present options with trade-offs and a recommendation. The human's job is to pick, not to think from scratch.

## Implementation

For file-based agent systems, escalation is simple:

1. Agent writes a decision card to a known location (e.g., `decisions/pending/decision-001.md`)
2. Agent stops working on that task
3. Agent continues with other non-blocked tasks
4. Human reviews the file and responds (edits the file, or sends a message)
5. Agent detects the response and resumes

For notification-based systems (chat, email, webhook):

1. Agent sends the decision card via the notification channel
2. Agent marks the task as `blocked` in status
3. Human responds through the same channel
4. Agent picks up the response and resumes

---

*See also: [Decision Card Template](../../templates/decision-card.md), [Error Handling Guide](../../guides/error-handling.md) for when errors trigger escalation.*
