# AGENT PROTOCOL

You are a stateless worker. Your memory is the `.ai/` directory. You have no recall of previous sessions except what is written there.

## PHASE 1: ORIENT (Mandatory before any code generation)
0. Read `.ai/README.md` — understand `.ai/` file roles, budgets, and compaction rules.
1. Read `.ai/CONTEXT.md` — acknowledge the tech stack and conventions (prefer links over duplication).
2. Read `.ai/TASKS.md` — identify the active task by status.
   - If multiple tasks are marked active, prefer the first task that is **IN PROGRESS**; if none are in progress, ask before choosing.
3. Read `.ai/LOG.md` (last 5 entries) — note recent failures and learnings.
4. Read `.ai/DECISIONS.md` — confirm no architectural constraints block your approach.
5. Read `.ai/STATE.md` (if present) — resume the current working set; update it with today’s session goal.
6. State your plan in ≤3 sentences. Wait for approval before proceeding.
   - If the user asked for review/advice only and you will not modify any files, stop after PHASE 1 and provide the review (PHASE 2/3 are not required).

## PHASE 2: EXECUTE
- Touch only files relevant to the active task.
- If you encounter an architectural question not covered by DECISIONS.md, STOP and ask.
- Run the specified test command after changes.
  - If the active task includes a `Test:` command, run that.
  - If no `Test:` command is specified, run `npm run lint` and `npm run build` (or ask if unclear).

## PHASE 3: UPDATE (Mandatory before session ends)
You may not consider a task complete until:
1. Test output is shown and passes.
2. TASKS.md status is updated.
3. LOG.md has a new entry with: action, result, reverted (y/n), learning.
4. If you made an architectural choice, append to DECISIONS.md.
5. If any `.ai/` file exceeds its budget, compact per `.ai/COMPACTION.md` and archive to `.ai/archive/YYYY-MM/`.
6. Update `.ai/STATE.md` to reflect the handoff (or reset it to the template for the next session).

## FAILURE MODE
If tests fail after 2 attempts on the same approach:
1. Revert to last working state.
2. Log the failure with root cause hypothesis.
3. Mark task as BLOCKED with reason.
4. Stop and surface the blocker to the user.

## PROJECT PATTERN RULES (Advizor Connect)
- `server/src/routes/v1.ts` is the v1 route **composer**; add new route groups as modules under `server/src/routes/v1/*.ts` and register them from `registerV1Routes`.
- Any DB mutation spanning multiple statements runs in a transaction and avoids “rollback after commit” by tracking a `committed` boolean.
- Idempotency/concurrency: for create-or-reuse endpoints, prefer Postgres `pg_advisory_xact_lock(hashtext(<stable key>))` around the critical section.
- Email delivery: use Resend `Idempotency-Key` for retry-safety and persist `last_sent_at` / provider message ids for observability + cooldown enforcement.
