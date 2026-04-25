# Completion Notification

> Every agent must report "done" or "failed" immediately and explicitly when a task finishes. This is a mandatory protocol, not a courtesy.

**Category**: Communication | **Complexity**: Low | **Prerequisite**: None

---

## Problem

You ask an agent to install a dependency, run tests, and update a config file. The agent does all three things. Then it writes a 500-word analysis of the test results. Somewhere in paragraph 3, it mentions "by the way, the config update is done."

You read the first paragraph, don't see a status, and ask: "Is it done?" The agent responds: "Yes, as I mentioned above..."

This wastes time and creates ambiguity. Multiply by 10 tasks per day and you have a real productivity drain.

## Solution

**Rule: Every operation completion gets its own explicit notification, separate from any analysis or discussion.**

### Format

| Outcome | Format |
|---------|--------|
| Success | `Task {task_id} completed. Output: {location}` |
| Failure | `Task {task_id} FAILED: {one-line reason}` |
| Partial | `Task {task_id} partially completed: {what worked}, {what didn't}` |

### Timing

- Send **immediately** upon task completion
- Send **before** any analysis or commentary
- Send as a **separate message**, not embedded in a longer response

### Examples

Good:
```
Task dependency-install completed. Installed pandas 2.1.0 and numpy 1.26.0.
```
```
Task run-tests FAILED: 3 of 47 tests failing. See artifacts/test-results.md for details.
```

Bad:
```
I analyzed the test results and found some interesting patterns. The coverage
improved by 12% compared to last run, particularly in the auth module. Also,
I should mention that the installation completed successfully and all tests
are now passing. Looking at the specific improvements...
```

(Where's the status? Buried in paragraph 1 of a long analysis.)

## Why This Pattern Exists

In practice, AI agents exhibit a consistent behavior: they complete a task and then immediately start analyzing or discussing the result. The completion status gets woven into the narrative rather than stated upfront.

This causes:
1. **Ambiguity** -- the human isn't sure if the task is actually done
2. **Missed failures** -- a failure mentioned in passing gets overlooked
3. **Blocked workflows** -- the orchestrator waits for a clear signal that never comes
4. **Follow-up waste** -- "Is it done?" "Did it work?" back-and-forth

## For Orchestrators

When you delegate to a subagent, include this in the task envelope:

```markdown
**Output Protocol**: When done, your FIRST line must be one of:
- "COMPLETED: {summary}" if successful
- "FAILED: {reason}" if unsuccessful
- "BLOCKED: {reason}" if you cannot proceed
Then provide any analysis below.
```

## For Multi-Step Tasks

Long tasks should send progress notifications at milestones, not just at the end:

```
Task data-pipeline: Step 1/4 completed (data fetched, 15K rows)
Task data-pipeline: Step 2/4 completed (cleaned, 14.2K rows after dedup)
Task data-pipeline: Step 3/4 FAILED (analysis script crashed: MemoryError)
```

This is especially important for tasks that might take more than 5 minutes. Without progress updates, the human can't tell the difference between "working" and "stuck."

---

*See also: [Structured Error Events](structured-error-events.md) for the full status format specification.*
