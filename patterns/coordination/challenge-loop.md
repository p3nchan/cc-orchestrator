# Challenge Loop

> When one agent reviews another's output, politeness is the default failure mode. A challenge loop forces review to produce evidence, not vibes.

**Category**: Quality Control | **Complexity**: High | **Prerequisite**: [Task Envelope](task-envelope.md)

---

## Problem

You ask Agent B to review Agent A's work. Agent B says "looks good" or raises vague concerns like "this might be risky." Nothing is falsifiable. Nothing changes. The system feels rigorous because two agents looked at the same artifact, but the review loop is mostly theater.

This gets worse when both agents share the same style, training priors, or desire to be agreeable. Research from SycEval in AAAI 2025 found sycophancy in 58% of LLM responses. The ELEPHANT benchmark later found LLMs were 45 percentage points more sycophantic than humans. If your review stage is not explicitly adversarial, it will drift toward rubber-stamping.

## Solution

Run review as a **Challenge Loop**:

```
Producer (Agent A) → Challenger (Agent B, different model family) → Judge (orchestrator)
```

The rule is simple: every challenge must come with **testable evidence**. No evidence, no challenge.

### Acceptable evidence

| Evidence Type | What It Looks Like |
|---------------|--------------------|
| **Failing test** | "This input should return 400. It returns 200." |
| **Reproducible counterexample** | "This strategy loses money when slippage is > 10 bps." |
| **Spec violation** | "Acceptance criterion 3 requires idempotency. The handler mutates state on retry." |
| **Measurable regression** | "Latency increased from 120ms to 480ms on the benchmark fixture." |

If the challenger cannot produce one of these, the orchestrator should treat the feedback as weak by default.

## How It Works

### 1. Use role separation on purpose

The producer is trying to finish the work. The challenger is trying to break it. The judge decides what matters.

Use a different model family for the challenger. Research from Huang et al. at ICLR 2024 found LLMs do not reliably self-correct reasoning without external signals. Self-Correction Bench later measured a 64.5% blind spot rate. Research from Young in 2026 adds the missing reason: debate has a phase transition. If both agents share the same knowledge base, the benefit collapses toward zero. Cross-family disagreement is the signal you want.

### 2. Make the judge decide after an independent read

Before reading the challenger's notes, the judge should form an independent assessment of the artifact. This is the easiest anti-sycophancy control. Otherwise, the judge gets anchored by the first critique it sees.

After that, classify each challenge:

| Verdict | Meaning | Action |
|---------|---------|--------|
| **VALID** | The evidence proves a real issue | Fix it |
| **PARTIAL** | The concern is directionally right, but the evidence is incomplete or overstated | Narrow the issue, then fix or re-test |
| **NOISE** | The challenge is not supported, not relevant, or already handled | Discard it |

### 3. Measure whether the challenger is doing real work

Track the **substantive challenge rate**:

```
VALID + PARTIAL challenges
--------------------------
Total challenges raised
```

If that rate falls below 20%, your challenge prompting is failing. The reviewer is generating noise instead of adversarial signal. Change the prompt. "List 3 concrete ways this will fail" works better than "review this."

### 4. Converge aggressively

Stop the loop when both are true:

1. One full round produces **zero VALID** challenges
2. All external tests pass

That is your convergence condition. Do not keep looping just because another round is available.

### 5. Cap the loop

Use two caps:

| Cap | Rule | Why |
|-----|------|-----|
| **Soft cap: 3 rounds** | Expect useful movement in the first 2-3 rounds | Research summarized in the MAST taxonomy found overthinking loops were 39.3% of failures in financial processing benchmarks |
| **Hard cap: 10 rounds** | Escalate to a human with the best current state and unresolved issues | Beyond this point, the loop is no longer quality control. It is churn. |

### 6. Escalate to Expert-Executor early

If the loop stalls at round 2, do not let the executor keep trying random fixes. Switch patterns:

```
Executor stalls → Senior agent diagnoses → Executor applies narrowed fix
```

ChatGPT deep research on failure recovery found the Expert-Executor pattern recovered 22.2% of previously failed issues, versus 7% for single-agent retry. When the loop stalls, the problem is usually strategic: bad spec, wrong abstraction, or the wrong test target.

## Anti-Patterns

### DOT (Degeneration-of-Thought)

Each round gets longer, more complicated, and less grounded in evidence.

**Fix**: Force the challenger to attach one testable artifact per claim.

### Mirror Loop

Producer and reviewer share the same model family and mostly restate each other in different words.

**Fix**: Switch reviewer family. Similar priors create similar blind spots.

### Rubber-stamping

The reviewer approves because the output looks polished.

**Fix**: Review against the spec and tests, not tone or confidence.

### Sycophantic challenge theater

The reviewer tries to sound critical but raises low-value objections that never survive validation.

**Fix**: Track substantive challenge rate. If it is under 20%, the review prompt needs to change.

## Real-World Example

A financial algorithm team used a challenge loop to pressure-test a six-round spec review:

| Round | VALID | NOISE | What Happened |
|-------|:---:|:---:|---------------|
| R1 | 3 | 7 | Challenger found a look-ahead leak, missing slippage model, and an unbounded retry path |
| R2 | 2 | 4 | Producer fixed the leak and slippage assumptions; reviewer found a purge-window bug |
| R3 | 1 | 2 | Only the benchmark split logic remained broken |
| R4 | 1 | 1 | Judge marked one claim PARTIAL and narrowed it to a single fixture |
| R5 | 0 | 1 | External tests passed except one stale scenario |
| R6 | 0 | 0 | External tests all green → ACCEPT |

The key was not "six rounds." The key was that every accepted challenge came with evidence the judge could test.

---

*See also: [Task Envelope](task-envelope.md), [Model Selection](../../guides/model-selection.md), [Error Handling](../../guides/error-handling.md)*
