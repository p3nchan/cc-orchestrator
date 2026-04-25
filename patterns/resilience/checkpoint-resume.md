# Checkpoint & Resume

> Before any high-risk operation or at the end of any significant phase, write a checkpoint. When a session crashes, you resume from the checkpoint instead of starting over.

**Category**: Resilience | **Complexity**: Medium | **Prerequisite**: [File Blackboard](../coordination/file-blackboard.md)

---

## Problem

You're 45 minutes into a complex multi-phase task. The session crashes (timeout, network issue, context window exhaustion). You start a new session. The new agent has no idea what happened. It either:
- Starts over from scratch (wasting all previous work)
- Reads stale files and makes wrong assumptions about current state
- Asks the human "where were we?" (wasting the human's time)

## Solution

Write periodic checkpoints to disk. A checkpoint captures the exact state of the task so any new session can pick up where the last one left off.

### Checkpoint Contents

```json
{
  "phase": 2,
  "step": "cross_validate",
  "completed": ["collect_data", "initial_analysis"],
  "next": "cross_validate_sources",
  "artifacts": [
    "artifacts/collected-data.md",
    "artifacts/initial-analysis.md"
  ],
  "pending_tasks": ["validate-source-a", "validate-source-b"],
  "cost_estimate_usd": 1.20,
  "updated_at": "2026-03-15T11:00:00Z"
}
```

### When to Checkpoint

| Trigger | Why |
|---------|-----|
| Phase or major sub-task completed | Natural save point with validated output |
| Before spawning a high-risk subagent | If the subagent fails catastrophically, you can roll back |
| Before any external API call | External calls can fail in unpredictable ways |
| Detection of impending interruption | Graceful shutdown |
| At least once per day for long-running work | Insurance against silent crashes |
| Context window getting full | Session may need to restart soon |

### When NOT to Checkpoint

| Situation | Why |
|-----------|-----|
| After every single tool call | Overhead exceeds value |
| In the middle of an atomic operation | Checkpoint would capture inconsistent state |
| When nothing has changed since last checkpoint | Redundant writes |

## Atomic Writes

Checkpoints must be written atomically to prevent corruption from crashes mid-write:

```
1. Write to checkpoint.tmp
2. Verify the file is valid JSON and complete
3. Rename checkpoint.tmp → checkpoint.json (atomic on most filesystems)
```

If step 3 never executes (crash during step 1 or 2), the old `checkpoint.json` remains intact. You never end up with a half-written checkpoint.

## Cold Start Boot Sequence

When a new session starts and finds existing work:

```
1. Check for checkpoint.json in the workspace
2. If found:
   a. Read the checkpoint
   b. Verify referenced artifacts exist
   c. Read the current status.md
   d. Synthesize a resume plan: "I was at Phase 2, Step: cross_validate.
      Completed: data collection, initial analysis.
      Next: cross-validate sources A and B."
   e. Continue from next step
3. If not found:
   a. Check status.md for any progress indicators
   b. If status shows in-progress work, assess what's salvageable
   c. If nothing useful, start fresh
```

## Checkpoint Lifecycle

Checkpoints are not permanent. They follow a lifecycle:

```
Created (task in progress)
  │
  │  Phase completed successfully
  ▼
Superseded (new checkpoint for next phase)
  │
  │  All phases completed
  ▼
Deleted (project marked complete, no need for recovery)
```

### Cleanup Rules

| Event | Action |
|-------|--------|
| Phase N+1 starts | Checkpoint for Phase N can be deleted |
| Project completed | All checkpoints deleted |
| Checkpoint older than 7 days with no activity | Review -- probably stale, archive or delete |

Don't accumulate checkpoints. Each project should have at most one active checkpoint at a time.

## Failure Modes

| Failure | Symptom | Prevention |
|---------|---------|------------|
| Checkpoint references deleted artifacts | Resume fails because `artifacts/analysis.md` no longer exists | Never delete artifacts while their checkpoint is active |
| Stale checkpoint | New session resumes from a week-old checkpoint, redoing work that was actually completed | Update or delete checkpoints whenever status changes |
| Corrupted checkpoint (non-atomic write) | JSON parse error on resume | Use atomic write (tmp + rename) |
| Missing checkpoint | Session crashed before first checkpoint | Checkpoint early -- at minimum, after the first completed sub-task |

## Multi-Phase Example

```
Phase 1: Data Collection
  └─ checkpoint: {phase: 1, step: "collect", completed: [], next: "fetch_api"}
  └─ ... agent fetches data ...
  └─ checkpoint: {phase: 1, step: "done", completed: ["fetch_api", "fetch_db"]}

Phase 2: Analysis
  └─ checkpoint: {phase: 2, step: "analyze", completed: ["collect"]}
  └─ ... agent runs analysis ...
  └─ SESSION CRASHES HERE

New session starts:
  └─ Reads checkpoint: {phase: 2, step: "analyze"}
  └─ Reads artifacts from Phase 1 (still on disk)
  └─ Resumes analysis from where it left off
```

The crash cost zero work because the checkpoint captured the state between phases.

---

*See also: [Checkpoint Template](../../templates/checkpoint.json), [Error Handling Guide](../../guides/error-handling.md) for rollback strategies.*
