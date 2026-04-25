# Context Management Guide

> Context is an operational risk, not just a prompting concern. Bloated context causes hallucinations, increases cost, and slows down every interaction. Manage it actively.

---

## Core Principle

**Subagents are stateless functions.** They receive input, produce output, and terminate. Every token in their prompt costs money and affects quality. Keep prompts lean.

## The Three-Layer Memory Model

| Layer | Scope | Size Target | Examples |
|-------|-------|:---:|---------|
| **Working Memory** | Current task | < 8K tokens | Task envelope, relevant file paths, acceptance criteria |
| **Project State** | Current project | Read on demand | status.md, context.md, recent artifacts |
| **Long-term Memory** | System-wide | Indexed, never loaded in full | Debug KB, past decisions, configuration |

### The key insight

Only Working Memory goes into a subagent's prompt. Project State is read by the subagent as needed (via file reads). Long-term Memory is only accessed when the orchestrator decides it's relevant.

**Never dump Project State or Long-term Memory into a subagent prompt.**

## What Goes in a Subagent Prompt

| Include | Exclude |
|---------|---------|
| Task goal (1 sentence) | Project history |
| Acceptance criteria | Other tasks' details |
| Input file paths + 1-line descriptions | Full file contents |
| Relevant constraints | System-wide configuration |
| Stop conditions | Previous failed attempts (unless specifically relevant) |
| Output format | The orchestrator's reasoning about why this task exists |

### Target: < 8K tokens for a subagent prompt

If your subagent prompt exceeds 8K tokens, you're probably including too much context. Check for:
- Full file contents that should be paths
- Historical context that's no longer relevant
- Redundant explanations

## Path-Passing Protocol

Instead of injecting file contents, pass paths with summaries:

```markdown
**Inputs**:
- `data/users.csv` — User table, 15K rows, columns: id, name, email, created_at
- `config/rules.json` — Validation rules, 12 rules covering email format and name length
- `tests/test_validation.py` — Existing test suite, 23 tests, all passing
```

The subagent reads the files it needs. Benefits:
- Files only enter context when actually needed
- The subagent can read selectively (e.g., only the first 100 lines)
- No stale data -- the subagent reads the current version

## Phase Compression

After each phase completes, compress it to a summary:

### Before compression

The orchestrator's context contains:
```
Phase 1 task envelope (200 tokens)
Phase 1 subagent output (1500 tokens)
Phase 1 validation results (300 tokens)
Phase 1 revision request (150 tokens)
Phase 1 revised output (1200 tokens)
Phase 1 final validation (200 tokens)
Total: ~3,550 tokens
```

### After compression

```
Phase 1 completed: Collected user data (15K rows), validated schema,
cleaned 847 duplicate rows. Output: artifacts/phase1-clean-data.csv
Total: ~30 tokens
```

### When to compress

- Immediately after a phase is validated and accepted
- Before starting the next phase
- When context window usage exceeds 50%

## Context Window Health Check

Monitor for these symptoms of context bloat:

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Agent repeats earlier reasoning | Context too long, model losing focus | Compress or restart |
| Agent forgets a constraint you stated earlier | Key info is too far back in context | Restate key constraints, or restart with fresh summary |
| Agent hallucinates details about a file | File contents were summarized, not read | Have agent re-read the actual file |
| Responses getting slower | Context approaching model limits | Compress and/or restart |

### The restart heuristic

If you notice any of the symptoms above, it's cheaper to:
1. Checkpoint current state
2. Start a fresh session
3. Load only: checkpoint summary + current task

Than to keep fighting a bloated context.

## Multi-Agent Context Isolation

Each subagent gets its own isolated context. This is a feature, not a limitation.

### Why isolation matters

- **Prevents contamination**: Agent A's failed attempt doesn't pollute Agent B's context
- **Enables parallel execution**: Agents don't need to share state
- **Limits blast radius**: If one agent hallucinates, it doesn't spread to others

### The orchestrator is the bridge

Information flows: Subagent A → Orchestrator → Subagent B

The orchestrator decides what to pass forward. It can:
- Filter out irrelevant details
- Correct errors before they propagate
- Translate between different agents' output formats

**Subagents never read each other's output directly.** The orchestrator reads, validates, and selectively passes relevant information.

## Anti-Patterns

### The kitchen sink prompt

"Here's the entire project context, all recent changes, the team guidelines, and the coding standards. Now write one function."

**Fix**: Include only what's needed for this specific task. If the agent needs more, it can ask or read files.

### The accumulator session

A session that runs for 100+ turns, carrying all historical context. By turn 80, the model is confused and slow.

**Fix**: Long sessions need periodic compression. Checkpoint, summarize, restart fresh. Think of it as garbage collection for context.

### The duplicate context

The same information appears in the system prompt, the task envelope, and a file the agent reads. Triple the tokens, no benefit.

**Fix**: State each piece of information once, in the most appropriate location.

---

*See also: [Task Envelope](../patterns/coordination/task-envelope.md) for structuring inputs, [Cost Control](cost-control.md) for the financial impact of context bloat.*
