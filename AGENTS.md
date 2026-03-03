# AGENT PROTOCOL

You are a stateless worker. Your memory is the `.ai/` directory. You have no recall of previous sessions except what is written there.

## DIRECTORY STRUCTURE

```
.ai/
  planned/          ← scoped tasks, not yet started
  in-progress/      ← actively being worked on
  verifying/        ← code complete, verification criteria must pass
  done/             ← verified and closed
    2026-03/        ← archived by month
  blocked/          ← stuck, reason in file
  README.md         ← .ai/ file roles, budgets, compaction rules
  CONTEXT.md        ← tech stack, conventions, project-specific constraints
  DECISIONS.md      ← architectural choices and rationale
  LOG.md            ← structured session log
  STATE.md          ← current session handoff state
```

**State transitions are file moves:**
`planned/` → `in-progress/` → `verifying/` → `done/YYYY-MM/`
Also: any state → `blocked/` · Move back to `planned/` when unblocked.

If verification fails in `verifying/`: move back to `in-progress/` for rework (or to `blocked/` if stuck after 2 attempts).

## TASK FILES

Each task is a standalone `.md` file named `T-NNN-short-slug.md`.

**Small tasks** — title, priority, and optional notes are enough:

```markdown
# T-012: Fix off-by-one in pagination
Priority: Low | Size: S
```

**Medium/Large tasks** — require a full spec:

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
1. Run consumer against staging queue — no errors in logs.
2. Throughput ≥ 200 docs/min sustained.
3. `npm run test -- --grep pipeline`

## Notes
%% backpressure strategy TBD — check queue depth limits
```

**`%%` annotations** — open questions left in specs for later resolution. Address all `%%` items before moving a task to `verifying/`.

## PHASE 1: ORIENT (Mandatory before any code generation)

0. Read `.ai/README.md` — understand `.ai/` file roles, budgets, and compaction rules.
1. Read `.ai/CONTEXT.md` — acknowledge the tech stack, conventions, and project constraints.
2. `ls` the task folders — identify what's in `in-progress/`, then check `planned/`.
   - If multiple tasks are in `in-progress/`, prefer the first by ID number; if unclear, ask before choosing.
   - Read the active task file to understand scope.
3. Read `.ai/LOG.md` (last 5 entries) — note recent failures and learnings.
4. Read `.ai/DECISIONS.md` — confirm no architectural constraints block your approach.
5. Read `.ai/STATE.md` (if present) — resume the current working set; update it with today's session goal.
6. State your plan in ≤3 sentences. Wait for approval before proceeding.
   - If the user asked for review/advice only and you will not modify any files, stop after PHASE 1 and provide the review (PHASE 2/3 are not required).

## PHASE 2: EXECUTE

- Touch only files relevant to the active task.
- If you encounter an architectural question not covered by DECISIONS.md, STOP and ask.
- Run the specified test/verify command after changes.
  - If the active task includes a `Verify:` section, run that.
  - If no verify section is specified, run `npm run lint` and `npm run build` (or ask if unclear).
- **Checkpoint long sessions:** if context is getting large, save progress to `.ai/STATE.md` before continuing. Compaction can drop relevant context — don't rely on it.

## PHASE 3: UPDATE (Mandatory before session ends)

You may not consider a task complete until:
1. All `Verify:` criteria pass (or tests pass if no criteria specified).
2. Task file is moved to the appropriate folder (`verifying/` or `done/YYYY-MM/`).
3. LOG.md has a new entry with: action, result, reverted (y/n), learning.
4. If you made an architectural choice, append to DECISIONS.md.
5. If any `.ai/` file exceeds its budget, compact per `.ai/COMPACTION.md` and archive.
6. Update `.ai/STATE.md` to reflect the handoff (or reset it to the template for the next session).

## FAILURE MODE

If tests fail after 2 attempts on the same approach:
1. Revert to last working state.
2. Log the failure with root cause hypothesis.
3. Move task file to `blocked/` — add a `## Blocked` section with the reason.
4. Stop and surface the blocker to the user.

## CRITICAL INVARIANTS

Project-specific safety constraints live in `.ai/CONTEXT.md`. The following are **hard rules** that apply regardless — always check CONTEXT.md for the full set:

- Architectural questions not in DECISIONS.md → STOP and ask.
- Never skip verification to move a task to `done/`.
- Never modify `.ai/` metadata files outside of Phase 1 (orient reads) and Phase 3 (session-end updates), **except** `STATE.md` which may be updated mid-session as a checkpoint. Task file moves are allowed during Phase 3.
- All `%%` annotations must be resolved before a task enters `verifying/`.
