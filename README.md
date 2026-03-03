# Agent Protocol

A lightweight protocol for AI coding agents that treats your filesystem as a kanban board.

## The Problem

AI agents are stateless — each session starts from zero. They need a way to pick up where they left off, understand what's been decided, and know what to work on next. Most approaches stuff everything into a single tasks file or a bloated CLAUDE.md. This doesn't scale when agents complete entire features in one session.

## The Approach

A `.ai/` directory in your project root acts as the agent's memory. Tasks are standalone markdown files that physically move between status folders — like cards on a kanban board. State transitions are file moves, not field edits.

```
.ai/
  planned/            Tasks scoped but not started
  in-progress/        Actively being worked on
  verifying/          Code complete, verification pending
  done/2026-03/       Verified and archived by month
  blocked/            Stuck with documented reason
  README.md           Directory roles and budgets
  CONTEXT.md          Tech stack and project constraints
  DECISIONS.md        Architectural choices
  LOG.md              Structured session history
  STATE.md            Session handoff state
```

## How It Works

The agent follows three phases every session:

1. **Orient** — read the `.ai/` directory to understand context, find the active task, check for blockers
2. **Execute** — implement the task, run verification
3. **Update** — move the task file, log the session, update handoff state

## Task Sizing

Not every task needs a full spec. The protocol scales with complexity:

- **Small** — a title and priority (`T-012: Fix off-by-one in pagination`)
- **Medium/Large** — full spec with Problem, Solution, Files, and Verify sections

## Key Design Decisions

**Folder-as-status over table-in-file.** Moving a file is atomic and unambiguous. No table formatting to corrupt, no merge conflicts when parallel agents work different tasks, and `ls .ai/in-progress/` is all an agent needs to find its work.

**Conditional spec depth.** Small fixes don't need a problem statement. Large features do. The protocol requires specs only for M/L tasks, keeping small work fast.

**Explicit verification.** Tasks can't reach `done/` without passing their `Verify:` criteria. If verification fails, the task moves back to `in-progress/` — no ambiguity about rework flow.

**Structured failure.** Two failed attempts on the same approach triggers a revert-and-block protocol. Agents don't thrash.

**Session logging with learnings.** Every session appends to LOG.md with action, result, reverted (y/n), and a learning field. This builds institutional memory that future sessions can draw from.

## Usage

Copy `AGENTS.md` into your project and reference it from your agent's system prompt or CLAUDE.md. Then create the `.ai/` directory structure:

```sh
mkdir -p .ai/{planned,in-progress,verifying,done,blocked}
```

Populate `.ai/README.md`, `CONTEXT.md`, and `DECISIONS.md` for your project, then create task files in `planned/` and point your agent at the work.

## Influences

This protocol evolved from comparing two approaches:
- A single-agent `.ai/` directory protocol with structured logging, failure modes, and compaction
- Manuel Schipper's [Feature Design system](https://schipper.ai/posts/parallel-coding-agents/) for parallel agent orchestration with rich specs and slash commands

The result keeps the simplicity and safety of a single-agent protocol while borrowing the spec-driven planning, explicit verification, and filesystem-as-state patterns that work well at scale.
