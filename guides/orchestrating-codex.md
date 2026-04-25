# Orchestrating Codex from Claude Code

> Stub — full guide is being written in a follow-up phase of the cc-orchestrator restructure (April 2026). When complete, this file will replace the stub with the synthesis of the Codex CLI 0.124 operations material plus the 4-Lane model. Track progress in the cc-orchestrator handoff notes.

This guide will cover:

- Picking the right lane for a given task: local `codex exec` · git worktree parallel · Codex Cloud · MCP + hooks + JSONL.
- Effort tier selection (`low` / `medium` / `high` / `xhigh`) and the credit-window math.
- Sandbox mechanics: when to flip `network_access=true`, when to keep it locked down, what `--full-auto` actually means.
- Observability: the `exit 0` + `turn.completed` + sidecar three-signal rule, JSONL parsing, stuck heuristics.
- Hooks (`SessionStart` / `UserPromptSubmit` / `PreToolUse` / `PermissionRequest` / `PostToolUse` / `Stop`), and the file-write paths where they don't fire (`apply_patch`).
- Failure recovery: the `/compact` timeout trap, orphan process cleanup, when to fork vs. resume vs. start fresh.
- The Top 10 pits: a triaged shortlist with failure signature → recovery.

For the cheatsheet form while you wait, see [README.md](../README.md) and [patterns/coordination/task-envelope.md](../patterns/coordination/task-envelope.md).
