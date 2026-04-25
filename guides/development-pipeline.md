# Development Pipeline Guide

> Multi-agent development needs phases, not vibes. But every phase must add genuinely new information, or you are paying overhead for the appearance of rigor.

---

## Problem

Teams adopt multi-agent development because it sounds safer: spec, review, fix, review again, maybe another reviewer, maybe parallel agents. The problem is that each stage has a failure rate and a cost. Add too few phases and important issues survive. Add too many and the pipeline collapses under coordination overhead.

That trade-off is measurable. DeepMind's orchestration scaling work found parallel agents improved parallelizable tasks by 80.9%, but hurt sequential reasoning by 39% to 70%. The same line of work showed communication overhead scaling at an exponent of 1.724, which is close to quadratic. More phases and more agents only help when they inject new information.

## The Operating Principle

**Each step must introduce genuinely new information. A step that doesn't add new info is overhead, not rigor.**

That means:

- spec clarifies intent
- challenge finds contradictions
- implementation creates the artifact
- review adds independent judgment
- validation adds external evidence

If two adjacent phases are reasoning from the same artifact with the same priors, collapse them.

## How It Works

### Algorithm Development Pipeline

Use the full 5-phase pipeline when the work is strategy-like, high-stakes, or expensive to mis-validate.

```
Phase 0   → Problem Definition
Phase 0.5 → External Validation (conditional)
Phase 1   → Spec + Challenge
Phase 2   → Implementation + Validation
Phase 3   → Cross-family Review + Fix
Phase 4   → External Challenge (conditional)
Phase 5   → Incubation → Production → Monitoring
```

### Phase 0: Problem Definition

Start mechanism-first. As Marcos Lopez de Prado puts it, backtest is monetization, not discovery.

Use a **Hypothesis Card**:

- falsifiable statement
- kill criteria
- validation timeline
- trial budget

If those fields are missing, you are probably optimizing after the fact.

### Phase 0.5: External Validation

This phase is conditional. Use it when the direction is uncertain or the downside of pursuing the wrong thesis is high. Run deep research in parallel before committing to the spec.

### Phase 1: Spec + Challenge

Build a Structured Handoff, then run a [Challenge Loop](../patterns/coordination/challenge-loop.md). This is where you remove the most expensive misunderstandings: data leakage, wrong objective, invalid benchmark, and vague success criteria.

### Phase 2: Implementation + Validation

Implementation is only half of this phase. Validation must climb a ladder:

1. walk-forward
2. bootstrap
3. PBO
4. synthetic stress

If the strategy only survives one validation lens, you do not know enough yet.

### Phase 3: Cross-family Review + Fix

Skip self-review. Use a different model family and keep the loop short. Hard cap the fix cycle at 3 productive rounds. If review stops adding evidence, escalate.

### Phase 4: External Challenge

This is also conditional. Use it for capital-allocation decisions, novel strategies, or systems where hidden assumptions can create large real-world losses.

### Phase 5: Incubation → Production → Monitoring

Use a three-bucket system:

| Bucket | Meaning |
|--------|---------|
| **Active** | Currently deployed and monitored |
| **Bench** | Passed research validation but not promoted |
| **Retired** | No longer trusted or economically useful |

Promotion should be based on predefined criteria, not enthusiasm after a good backtest.

### Code Development Pipeline

For normal software work, use a shorter 4-phase pipeline:

```
Phase 0 → Spec
Phase 1 → Spec Challenge (conditional, complex tasks only)
Phase 2 → Implementation
Phase 3 → Cross-family Review + Fix + Accept
```

This is enough because code usually has tighter feedback loops than algorithm research. If the task is simple and easily testable, you can skip Phase 1. If the task is complex or interface-heavy, keep it.

### When Orchestration Helps

Parallel agents are not a default. They are a fit decision.

| Situation | Use Orchestration? | Why |
|----------|--------------------|-----|
| **3+ independent modules** | Yes | DeepMind measured an 80.9% gain on parallelizable work |
| **Sequential reasoning chain** | Usually no | The same research saw 39% to 70% degradation |
| **Shared dependency bottleneck** | Usually no | Agents mostly wait or duplicate work |
| **Exploration with deterministic scoring** | Yes | Multiple candidates can be evaluated in parallel |

Practical limit: keep it to 3-4 agents. Beyond that, coordination cost usually dominates. A 4-agent pipeline can consume roughly 3.5x the tokens of a single-agent flow once communication and verification are included.

### Visual Phase Diagram

```
Algorithm work:
  Problem → Validate direction? → Spec → Challenge → Build → Validate → Review → External challenge? → Incubate → Prod

Code work:
  Spec → Challenge? → Build → Review → Accept
```

If you want to turn this into an asset, keep the contrast simple: long pipeline for algorithm work, short pipeline for code work, with conditional gates clearly marked.

## Anti-Patterns

### Ad-hoc "just build it"

The agent starts coding before the task is bounded.

**Fix**: Write the smallest spec that makes success testable.

### Too many ceremonial phases

The system adds review, re-review, post-review, and extra judges without adding new evidence.

**Fix**: Collapse phases that reason from the same artifact and priors.

### Parallelizing sequential work

Multiple agents work on a reasoning chain that only one agent can progress at a time.

**Fix**: Orchestrate only when modules are genuinely independent.

## Real-World Example

A team building a pricing engine used the short code pipeline for the API layer and the long algorithm pipeline for the pricing logic:

1. The API task got a medium-complexity spec, one implementation pass, and one cross-family review.
2. The pricing model got a hypothesis card, external validation on assumptions, a challenged spec, and a full validation ladder.
3. The team ran parallel agents only on independent fixtures and reporting modules.
4. They avoided a 5-agent orchestration plan because most of the logic was sequential and would have paid overhead without extra signal.

That split is the point: use the light pipeline for code, the heavy pipeline for research-like algorithm work, and do not confuse the two.

---

*This guide synthesizes findings from four parallel deep research tracks completed on April 15, 2026.*

*See also: [Challenge Loop](../patterns/coordination/challenge-loop.md), [Code Review](code-review.md), [Spec-Driven Development](spec-driven-development.md), [Cost Control](cost-control.md)*
