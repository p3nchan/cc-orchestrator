# Cost Control Guide

> Every task has a budget. Every retry costs money. Every bloated prompt wastes tokens. Cost control is not optional -- it's the difference between a sustainable system and one that bankrupts you in a week.

---

## Three Levels of Cost Control

| Level | Mechanism | Impact |
|-------|-----------|--------|
| **Per-task** | Budget field in task envelope (max tool calls, max tokens) | Prevents runaway individual tasks |
| **Per-project** | Daily/weekly cap tracked in status.md | Prevents any single project from dominating spend |
| **System-wide** | Circuit breaker + model selection + context pruning | Reduces baseline cost across everything |

## Per-Task Budgets

Every task envelope should include a budget:

```markdown
- **Budget**: max 10 tool calls, ~$0.50 estimated
```

### Estimating task cost

| Task Type | Typical Tool Calls | Typical Cost (Sonnet) | Typical Cost (Opus) |
|-----------|:---:|:---:|:---:|
| File read + analysis | 3-5 | $0.05-0.10 | $0.30-0.50 |
| Code write + test | 5-10 | $0.10-0.25 | $0.50-1.00 |
| Multi-file refactor | 10-20 | $0.25-0.50 | $1.00-3.00 |
| Research (web + synthesis) | 5-15 | $0.15-0.40 | $0.75-2.00 |
| Review + feedback | 3-5 | $0.05-0.10 | $0.30-0.50 |

These are rough estimates. Track your actual costs and adjust.

### What happens when budget is exceeded

```
Task hits budget limit →
  1. Stop executing (don't squeeze in "just one more" call)
  2. Save whatever progress exists
  3. Report to orchestrator: "Budget exhausted at step N of M"
  4. Orchestrator decides: allocate more budget, accept partial result, or escalate
```

## Token Budget as a Health Signal

A budget is not just a spending limit. It is also a diagnostic signal.

Research from the OpenHands SWE-bench work found higher token usage correlated with **lower** accuracy. Expensive runs were often failing runs that kept retrying. METR reported agent performance plateauing around 200K tokens regardless of budget. Even GPT-5.1-Codex-Max with a 32M-token budget showed only modest productive use beyond 5M tokens.

The practical lesson: once a task blows past its expected budget, treat that as a **stuck signal**, not as evidence the agent is "almost done."

### Budget by task tier

| Task Tier | Expected Token Shape | Stuck Signal |
|-----------|----------------------|--------------|
| **Simple** | Tens of thousands | Exceeds 2x plan |
| **Medium** | Low six figures | Exceeds 1.5x plan |
| **Complex** | Up to ~200K before checkpointing | Keeps growing without new artifacts |

If the budget is exceeded, stop and ask why. The answer is usually bad decomposition, weak specs, or a fix loop that should have escalated.

## Context Pruning

The single biggest cost driver is **bloated prompts**. Every token in the prompt costs money on every turn.

### The #1 Rule: Pass Paths, Not Content

Bad (costs tokens every turn):
```
Here is the full contents of the config file:
{... 500 lines of JSON ...}
Analyze this config.
```

Good (agent reads the file itself, one-time cost):
```
Read the config at `config/settings.json` (application settings, ~500 lines).
Focus on the database connection section.
```

### Token savings from path-passing

| Approach | Input Tokens | Savings |
|----------|:---:|:---:|
| Full file in prompt (500 lines) | ~2,000 | -- |
| Path + one-line summary | ~30 | **98%** |
| Path + relevant section hint | ~50 | **97%** |

The agent reads the file anyway -- but the file content only enters context once (when the agent reads it), not on every subsequent turn.

### Phase compression

After a phase is complete, compress it:

```
Before (in context):
  - Full task envelope for Phase 1
  - All Phase 1 tool call results
  - Phase 1 analysis
  - Phase 1 validation
  Total: ~5,000 tokens

After compression:
  - "Phase 1 complete: collected 15K rows of market data,
     validated schema, output at artifacts/phase1-data.csv"
  Total: ~30 tokens
```

### When to start a fresh session

If your context window is more than 60% full, consider:
1. Writing a checkpoint
2. Compressing all completed work into a summary
3. Starting a fresh session that loads only the summary + current task

This is cheaper than continuing with a bloated context, because every subsequent tool call pays for all that historical context.

## Anti-Patterns That Burn Money

### 1. Retry storms

Three retries of an Opus call = 4x the cost of the original call (original + 3 retries). If the error is not transient, you're burning money for nothing.

**Fix**: Classify errors. Only retry transients. See [Error Handling](error-handling.md).

### 2. Using Opus for everything

"I want the best results."

**Fix**: Use the [Model Selection](model-selection.md) two-tier rule. Opus for judgment, Sonnet/Haiku for execution.

### 3. Spawning agents for trivial tasks

"I'll spawn a subagent to read this file and summarize it."

**Fix**: If the orchestrator can do it in one tool call, don't spawn. Spawning has overhead (new context, system prompt, etc.).

### 4. Not compressing completed phases

The agent carries the full history of Phase 1, 2, and 3 in context while working on Phase 4.

**Fix**: Compress completed phases to one-line summaries. Only the current phase needs full detail.

### 5. Unlimited parallel spawns

Spawning 5 agents in parallel seems efficient. But if they all hit the same API, you get rate-limited, they all retry, and you pay 5x the retry cost.

**Fix**: Cap parallel spawns at 2-3. This avoids rate limit cascades and keeps cost predictable.

### 6. No budget in task envelopes

The task has no budget limit. The agent explores for 30 tool calls when 10 would have sufficed.

**Fix**: Always set a max tool calls in the task envelope. If the agent needs more, it should report back, not keep going.

## Cost Tracking

Track costs at the project level. A simple approach:

```markdown
## Cost Log — Project X

| Date | Task | Model | Est. Cost | Notes |
|------|------|-------|-----------|-------|
| 03-15 | research-api | sonnet | $0.15 | 7 tool calls |
| 03-15 | review-code | opus | $0.80 | 4 tool calls |
| 03-15 | fix-tests | sonnet | $0.10 | 5 tool calls |
| **Total** | | | **$1.05** | |
| **Budget** | | | **$5.00/week** | |
```

### Alerts

| Condition | Action |
|-----------|--------|
| Daily spend > 50% of weekly budget | Notify human |
| Single task > 150% of estimated cost | Stop + escalate |
| Weekly budget exhausted | Gate all new tasks until next week |

## Orchestration Overhead

Multi-agent systems have real coordination tax. Research from DeepMind found communication overhead scaling at an exponent of 1.724, which is close to quadratic. In practice, a 4-agent pipeline can cost roughly 3.5x the tokens of a single-agent flow once coordination is included. SoftwareSeni reported 40-50% coordination overhead in execution time, and AgentTaxo's analysis of systems like MetaGPT, CAMEL, and AgentVerse found verification phases consuming 72% of all tokens.

There is also an information-theory trap here. Under equal token budgets, single-agent systems often match or outperform multi-agent systems on multi-hop reasoning because communication loses information instead of creating it.

### Practical rules

| Rule | Why |
|------|-----|
| **Max 3-4 agents** | Diminishing returns kick in fast |
| **Only parallelize independent work** | Shared dependencies turn agents into overhead |
| **Treat verification as the main cost center** | In many systems, review burns more tokens than execution |
| **If one agent can finish in-budget, do that** | Orchestration is not free rigor |

## The 80/20 of Cost Optimization

These four changes cover ~80% of possible savings:

1. **Model selection** (expensive for judgment, cheap for execution) → 40-60% savings
2. **Path-passing instead of content injection** → 20-30% fewer input tokens
3. **Phase compression** → 15-25% fewer tokens in long sessions
4. **Budget limits in task envelopes** → prevents outlier costs

Everything else (caching, batching, fine-tuning) is optimization beyond the basics.

---

*See also: [Model Selection](model-selection.md), [Task Envelope](../patterns/coordination/task-envelope.md), [Context Management](context-management.md)*
