# AGENT PROTOCOL

You are a stateless worker. Your memory is the `.ai/` directory. You have no recall of previous sessions except what is written there.

## DIRECTORY STRUCTURE

```
.ai/
  planned/          <- scoped tasks, not yet started
  in-progress/      <- actively being worked on
  verifying/        <- code complete, awaiting human review
  done/             <- verified and closed
    YYYY-MM/        <- archived by month (e.g. 2026-03/)
  blocked/          <- stuck, reason in file
  README.md         <- .ai/ file roles, budgets, compaction rules
  CONTEXT.md        <- tech stack, conventions, project-specific constraints
  DECISIONS.md      <- architectural choices and rationale
  LOG.md            <- structured session log
  STATE.md          <- current session handoff state
```

**State transitions are file moves:**
`planned/` -> `in-progress/` -> `verifying/` -> `done/YYYY-MM/`
Also: any state -> `blocked/` . Move back to `planned/` when unblocked (clear or update the `## Blocked` section first).

If verification fails in `verifying/`: move back to `in-progress/` for rework (or to `blocked/` if stuck after 2 attempts).

### Who moves tasks between states

| Transition | Who |
|---|---|
| `planned/` -> `in-progress/` | Agent (Phase 1) |
| `in-progress/` -> `verifying/` | Agent (Phase 3, after self-verification passes) |
| `verifying/` -> `done/YYYY-MM/` | Human (after review) -- or agent if the task's `Verify:` section says `self-verify: true` |
| `verifying/` -> `in-progress/` | Human (review failed) or agent (verification criteria failed) |
| any -> `blocked/` | Agent or human |
| `blocked/` -> `planned/` | Human (after resolving the blocker) |

## CONCURRENCY

This protocol assumes **single-agent, single-session** execution. Do not run multiple agents or sessions against the same `.ai/` directory simultaneously. If parallel execution is needed, use separate worktrees with separate `.ai/` directories.

## TASK FILES

Each task is a standalone `.md` file named `T-NNN-short-slug.md`.

### Task ID allocation

To assign the next ID, scan all task files across all folders (`planned/`, `in-progress/`, `verifying/`, `done/`, `blocked/`) and use `max(NNN) + 1`. Never reuse IDs, even for deleted or archived tasks.

### Task creation

Tasks can be created by either the agent or the user. New tasks always start in `planned/`.

- **User-created:** User writes the task file directly into `planned/`.
- **Agent-created:** If the agent discovers subtasks or follow-up work during execution, it creates new task files in `planned/` and logs the creation in LOG.md. The agent does NOT auto-start these tasks -- finish the current task first.

### Task templates

**Small tasks** -- title, priority, and optional notes are enough:

```markdown
# T-012: Fix off-by-one in pagination
Priority: Low | Size: S
```

**Medium/Large tasks** -- require a full spec:

```markdown
# T-003: Streaming classification pipeline
Priority: High | Size: L

## Problem
Batch classification blocks the ingestion pipeline for 10+ minutes.

## Solution
Replace batch with streaming consumer pulling from queue in micro-batches of 50.

## Files
- src/pipeline/stream_consumer.py (new)
- src/pipeline/batch_consumer.py (modify: add graceful shutdown)
- sql/07_stream_state.sql (new)

## Verify
1. Run consumer against staging queue -- no errors in logs.
2. Throughput >= 200 docs/min sustained.
3. `npm run test -- --grep pipeline`

## Notes
%% backpressure strategy TBD -- check queue depth limits
```

**`%%` annotations** -- open questions left in specs for later resolution. Address all `%%` items before moving a task to `verifying/`.

## PHASE 1: ORIENT (Mandatory before any code generation)

0. Read `.ai/README.md` -- understand `.ai/` file roles, budgets, and compaction rules.
1. Read `.ai/CONTEXT.md` -- acknowledge the tech stack, conventions, and project constraints.
2. `ls` the task folders -- identify what's in `in-progress/`, then check `planned/`.
   - If multiple tasks are in `in-progress/`, prefer the lowest ID number (oldest first); if unclear, ask before choosing.
   - Read the active task file to understand scope.
3. Read `.ai/LOG.md` (last 5 entries) -- note recent failures and learnings.
4. Read `.ai/DECISIONS.md` -- confirm no architectural constraints block your approach.
5. Read `.ai/STATE.md` (if present) -- resume the current working set; update it with today's session goal.
6. State your plan in <=3 sentences. Wait for approval before proceeding.
   - If the user asked for review/advice only and you will not modify any files, stop after PHASE 1 and provide the review (PHASE 2/3 are not required).

## PHASE 2: EXECUTE

- Touch only files relevant to the active task. If the task has a `## Files` section, limit changes to those files unless you document the reason for touching others.
- If you encounter an architectural question not covered by DECISIONS.md, STOP and ask.
- Run the specified test/verify command after changes.
  - If the active task includes a `Verify:` section, run that.
  - If no verify section is specified, run the project's default verification (check `.ai/CONTEXT.md` for the command; common defaults: `npm run lint && npm run test && npm run build`).
- **Checkpoint long sessions:** if context is getting large, save progress to `.ai/STATE.md` before continuing.

### Commit policy

Commit after each task passes verification, before moving it to `verifying/` or `done/`. Use a conventional commit message referencing the task ID:

```
feat(T-003): add streaming classification pipeline
```

### Rollback procedure

If you need to revert changes:
- **Targeted:** `git checkout -- <files>` to restore specific files to their last committed state.
- **Preserve WIP:** `git stash` to save work-in-progress before reverting.
- **Full reset:** `git reset --hard HEAD` only as a last resort, and only if all work is committed or stashed.

## PHASE 3: UPDATE (Mandatory before session ends)

You may not consider a task complete until:
1. All `Verify:` criteria pass (or default verification passes if no criteria specified).
2. Changes are committed (see Commit policy above).
3. Task file is moved to `verifying/` (for human review) or `done/YYYY-MM/` (if `self-verify: true`).
4. LOG.md has a new entry with: action, result, reverted (y/n), learning.
5. If you made an architectural choice, append to DECISIONS.md.
6. If any `.ai/` file exceeds its line budget (defined in `.ai/README.md`), compact it: summarize older entries, archive the full version to `done/YYYY-MM/`, and keep the file within budget.
7. Update `.ai/STATE.md` to reflect the handoff or reset it for the next session:

```markdown
# STATE

## Current Task
None (or T-NNN if mid-task)

## Working Set
(files currently being modified)

## Session Goal
(what the next session should pick up)

## Open Questions
(anything unresolved)
```

## FAILURE MODE

If tests fail after 2 attempts on the same approach:
1. Revert to last committed state (see Rollback procedure).
2. Log the failure in LOG.md with root cause hypothesis.
3. Move task file to `blocked/` -- add a `## Blocked` section with the reason and failed approaches.
4. Stop and surface the blocker to the user.

## CRITICAL INVARIANTS

Project-specific safety constraints live in `.ai/CONTEXT.md`. The following are **hard rules** that apply regardless -- always check CONTEXT.md for the full set:

- Architectural questions not in DECISIONS.md -> STOP and ask.
- Never skip verification to move a task to `done/`.
- Never modify `.ai/` metadata files outside of Phase 1 (orient reads) and Phase 3 (session-end updates), **except** `STATE.md` which may be updated mid-session as a checkpoint. Task file moves are allowed during Phase 3.
- All `%%` annotations must be resolved before a task enters `verifying/`.
- Do not start new tasks while one is `in-progress` -- finish or block it first.
- Commit before moving a task out of `in-progress/`.
