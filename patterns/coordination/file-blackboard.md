# File Blackboard

> Agents communicate through files on disk, not through messages or shared memory. The filesystem is the message bus.

**Category**: Communication | **Complexity**: Low | **Prerequisite**: None

---

## Problem

AI coding tools (Claude Code, Codex, etc.) give you subagents that are:
- **Stateless** -- no memory between invocations
- **Isolated** -- no peer-to-peer messaging
- **Ephemeral** -- session can die at any time

You need a coordination mechanism that survives all of this.

## Solution

Use the filesystem as a shared blackboard. All agent state lives in files. The orchestrator is the only writer of global state; subagents write to designated output locations.

### Directory Layout

```
workspace/
  status.md              # Global project state (orchestrator-only writes)
  context.md             # Project context and background
  tasks/
    task-001.md          # Task envelope for subagent
    task-002.md
  artifacts/
    task-001-output.md   # Subagent output
    task-002-output.md
  checkpoints/
    checkpoint.json      # Crash recovery (see Checkpoint pattern)
```

### Access Rules

| Role | Read | Write |
|------|------|-------|
| Orchestrator | Everything | Everything |
| Executor subagent | Task envelope + referenced inputs | Own artifact file only |
| Reviewer subagent | Task envelope + artifact to review | Review notes only |
| Any subagent | -- | Never writes `status.md` or `context.md` |

## Why Files?

| Alternative | Problem |
|-------------|---------|
| In-memory shared state | Dies when any agent crashes |
| Message queues | Overkill for single-machine setups; adds infrastructure |
| Database | Overhead; AI agents work better with text files |
| Agent-to-agent API calls | Most platforms don't support this |

Files are:
- **Inspectable** -- you can `cat` any file to see current state
- **Durable** -- survive agent crashes, session resets, tool failures
- **Versionable** -- put the workspace in git for free audit trails
- **Universal** -- every AI coding tool can read and write files

## Write Discipline

The hardest part of File Blackboard is **write discipline**. Without it, you get:
- Race conditions (two agents writing the same file)
- State corruption (subagent overwrites orchestrator state)
- Orphaned artifacts (agent writes output but nobody reads it)

### Rules

1. **One writer per file.** If two agents need to contribute to the same document, they write separate files and the orchestrator merges.

2. **Orchestrator is the gatekeeper.** Subagent returns result -> orchestrator validates -> orchestrator writes to global state. Subagents never directly update `status.md`.

3. **Naming convention matters.** Use predictable paths so agents can find each other's work:
   ```
   artifacts/{task-id}-{agent-role}.md
   ```

4. **Atomic writes for critical state.** For checkpoint files, write to `.tmp` first, then rename:
   ```
   write checkpoint.tmp -> verify contents -> rename to checkpoint.json
   ```

## Scaling

| Agents | Approach |
|--------|----------|
| 1-3 | Single flat directory works fine |
| 4-7 | Subdirectories per project or phase |
| 8+ | Consider a `manifest.json` that indexes all active files |

## Failure Modes

| Failure | Symptom | Prevention |
|---------|---------|------------|
| Orphaned artifacts | Files exist but nobody reads them | Orchestrator must check for outputs after every subagent completes |
| Stale state | `status.md` says "in progress" but agent died | Checkpoints + periodic health checks |
| Path confusion | Subagent writes to wrong location | Use absolute paths in task envelopes |
| Encoding corruption | Binary/unicode issues in file content | Stick to UTF-8 markdown; validate after writes |

## Example

Orchestrator needs two subagents to research a topic in parallel:

```
1. Orchestrator writes:
   tasks/research-api.md     (Task envelope: "Research the REST API approach")
   tasks/research-graphql.md (Task envelope: "Research the GraphQL approach")

2. Orchestrator spawns Agent A with: "Read tasks/research-api.md, write output to artifacts/research-api.md"
   Orchestrator spawns Agent B with: "Read tasks/research-graphql.md, write output to artifacts/research-graphql.md"
   (Both run in parallel)

3. Both complete. Orchestrator reads:
   artifacts/research-api.md
   artifacts/research-graphql.md

4. Orchestrator synthesizes and updates:
   status.md  (marks research phase complete)
   context.md (adds key findings)
```

At no point did Agent A and Agent B communicate directly. The orchestrator coordinated everything through files.

---

*See also: [Task Envelope](task-envelope.md) for what goes in those task files, [Checkpoint & Resume](../resilience/checkpoint-resume.md) for crash recovery.*
