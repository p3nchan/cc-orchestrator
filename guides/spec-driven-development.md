# Spec-Driven Development Guide

> The ceiling on AI-generated code is usually the spec, not the model. If the brief is weak, better review stages only help you fail more carefully.

---

## Problem

Teams often optimize the visible stages of AI development: more review passes, more retries, more agents. But the code quality is still bounded by what the implementation agent was asked to build in the first place.

Research from ClarifyGPT showed spec refinement improved GPT-4 Pass@1 on MBPP from 70.96% to 80.80%, a 13.87% lift with p=3.2e-05. Research on progressive prompting found 96.9% task completion versus 80.5% for direct prompting, with a Cohen's d of 1.63. The pattern is consistent: stronger task framing beats extra downstream cleanup.

## The Core Idea

Treat the spec as the primary control surface.

That means:

- define what good looks like before implementation
- constrain what downstream agents can reinterpret
- attach validation targets that survive review

This repo calls that practice **Spec-Driven Development**. The exact phrase is newer than the underlying ideas, but the mechanics already show up in GitHub spec-kit, Anthropic Claude Code plan mode, and research systems that use structured handoffs.

## How It Works

### 1. Use a Structured Handoff Schema

Research on MetaGPT suggests its main advantage came from SOP-style structured outputs, not role-playing personas. The handoff format matters more than whether the agent is called "architect" or "senior engineer."

Use this schema:

| Field | Why It Exists |
|------|----------------|
| **Objective** | Prevents the task from drifting into adjacent work |
| **Context** | Gives the constraints the agent cannot infer |
| **Acceptance Criteria** | Defines "done" in a testable way |
| **Interface Contract** | Locks the boundaries that should not change accidentally |
| **Anti-patterns** | Names the failure modes to actively avoid |
| **Review Checklist** | Gives the reviewer a fixed target |

For algorithm work, add **Data Integrity Requirements**:

- no look-ahead leakage
- point-in-time data only
- purge + embargo splits where overlap can leak signal
- realistic transaction costs and slippage

These constraints are not detail. They are the difference between a plausible backtest and a fake edge.

### 2. Make acceptance criteria executable

This is the non-negotiable part.

The arXiv 2603.25773 review paper argues that AI code review without executable specifications is structurally circular. In experiments with domain-opaque bugs, review without specs ranged from 0% to 100% detection. With BDD-style scenarios used as the spec, detection reached 100%.

If the agent cannot prove it met the spec, you do not have a usable spec yet.

### 3. Scale the process to task complexity

Martin Fowler's 2026 guidance is useful here because it avoids process inflation:

| Task Complexity | Spec Shape |
|-----------------|-----------|
| **Simple** | Verbal spec + 1-2 acceptance criteria |
| **Medium** | Roughly one page + interface definition + test plan |
| **Complex** | Full Structured Handoff Schema |

The trap is over-specifying easy work. For medium tasks, a full SDD workflow can add more overhead than plain AI-assisted coding. Use the smallest spec that still prevents ambiguity.

### 4. Challenge the spec before you challenge the code

For complex work, run a spec challenge before implementation. That is not a separate academic technique with a single canonical name, but functionally similar practices keep appearing because they work: one agent proposes, another tries to break assumptions, a judge narrows the disagreement before execution begins.

The cheapest bug is the one removed before code exists.

### 5. Switch to an Evaluator Pattern when the right answer is unknown

Spec-Driven Development assumes you mostly know what "correct" looks like. If you are exploring algorithm design or search space discovery, use an **Evaluator Pattern** instead:

```
LLM proposes candidate → deterministic evaluator scores it → keep best → iterate
```

That is the core lesson from FunSearch in Nature 2024. The evaluator guards against confabulations because the model does not get to declare its own answer correct.

## Anti-Patterns

### Natural-language-only specs

The agent gets a broad paragraph and fills in the missing requirements by guessing.

**Fix**: Add acceptance criteria and an interface contract.

### Spec without acceptance criteria

The work sounds clear until review starts, then nobody agrees on what "done" means.

**Fix**: Write criteria that a test or scenario can actually verify.

### Over-specifying simple tasks

You spend more time filling templates than solving the problem.

**Fix**: Match the process to the complexity tier.

## Real-World Example

A team needed an agent to add a reconciliation endpoint to an internal finance tool.

The weak version of the brief was:

> "Add an endpoint to reconcile delayed settlements."

The Structured Handoff version was:

- **Objective**: Add a reconciliation endpoint for delayed settlements without mutating already-booked ledger rows
- **Context**: Existing system retries settlement callbacks and runs daily repair jobs
- **Acceptance Criteria**: Duplicate callbacks are idempotent, unmatched records are quarantined, and repair jobs emit audit events
- **Interface Contract**: Existing settlement schema cannot change
- **Anti-patterns**: No silent auto-merge, no test weakening, no hidden retry loop
- **Review Checklist**: Verify duplicate handling, audit trail, and rollback behavior

The implementation generated from the second spec took less review because the ambiguity had already been removed.

---

*See also: [Code Review](code-review.md), [Development Pipeline](development-pipeline.md), [Task Envelope](../patterns/coordination/task-envelope.md)*
