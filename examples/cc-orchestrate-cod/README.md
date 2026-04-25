# Example: Claude Code orchestrating Codex CLI

> Skeleton — minimal end-to-end example is being added in a follow-up phase. The shape it will demonstrate:
>
> 1. **Orchestrator script** (Claude Code session): reads a `task-envelope.md`, dispatches a `codex exec` worker, watches the JSONL stream, applies a circuit breaker on consecutive failures, emits a completion notification when done.
> 2. **Worker invocation**: a single Codex run against a small "fix this failing test" task, with `--output-last-message` sidecar and `--json` stream.
> 3. **Workspace**: `status.md` + `tasks/` + `artifacts/` + `checkpoints/` — the file blackboard pattern in action.
> 4. **Failure modes**: deliberate examples of what each pattern catches.
>
> Until then, the patterns and templates in this repo stand on their own. See [Task Envelope](../../templates/task-envelope.md) for the dispatch shape and [Circuit Breaker](../../patterns/resilience/circuit-breaker.md) for the failure-budget logic.
