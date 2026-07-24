---
name: phased-work-planner
description: >
  Plan and track long-running engineering work that spans multiple sessions. Use
  whenever the user (a) asks to plan a big effort before touching code — a rewrite,
  migration, port, service extraction, database engine change, large refactor, or a
  substantial new feature/module built from a referenced spec document; (b) asks to
  break work into phases or steps with per-step verification ("split this task up",
  "break it into phases", "plan it first, don't write code yet", "assess how to
  verify each step"); or (c) says the work can't finish in one sitting and will be
  advanced gradually over days or weeks. ALSO use whenever the user wants to resume
  or check previously saved plan progress — "resume plan", "continue where we left
  off", "continue the migration", "next phase", "move the work forward", "what's the
  current progress", or references a plan step id like S3. The plan and progress persist in
  `.work-planner/` and survive context resets. Skip for ordinary single-session
  work: a focused bug fix, small component tweak, README/copy edit, dependency
  bump, code explanation, or a casual "what's next" about the current conversation.
---

# Phased Work Planner

## What this skill is for

Driving any **multi-phase engineering effort** through the files that live in `<repo>/.work-planner/`:

| File | Role | What it answers |
|---|---|---|
| `summary.md` | Goal & context | Why this work exists, from-state → to-state, in/out of scope, assumptions, and the **Context & References** catalog |
| `plan.md` | Blueprint | The full step-by-step plan, dependencies, decisions, milestones. For very large efforts it becomes an index over `plan-<slug>.md` shards |
| `state.json` | Progress | Current step, what's done, what's next, what failed, timestamps, notes, per-step `refs` |
| `context/` | Raw inputs | The actual attachments the work was born from — screenshots, pasted API docs, error logs, spec excerpts — saved as files so they outlive the chat |

These files are the **single source of truth**. Everything else (this skill, the codebase, your memory) is secondary. If they disagree with anything you remember, **the files win**.

### Why `context/` exists

A plan is a *distillation* of the conversation. But the conversation that spawns it is usually full of raw material the distillation can't fully capture: a screenshot of the target UI, a pasted API contract, a stack trace, a spec paragraph, a reference URL. Screenshots and pasted text are **ephemeral** — they live only in the current chat, never on disk. A future session resuming from `.work-planner/` sees none of it, so it re-implements from a lossy summary and drifts far from what the user actually showed you.

The fix: whenever the triggering context carries reference material, **persist a durable copy into `.work-planner/context/` and catalog it in `summary.md`**, then point the steps that need it at those items via `refs`. This is what lets a cold session rebuild the same picture you have now. Treat missing context as a first-class failure mode, not a nicety.

Two modes, picked by reading the current state of `.work-planner/`:

- **Mode A — Bootstrap**: `.work-planner/` is missing or empty. You're starting a new plan. → Read `references/bootstrap.md` and follow it. Capturing context into `context/` is part of bootstrap, not an afterthought.
- **Mode B — Resume**: `.work-planner/` already has the state files. You're continuing an existing one. → Follow the "Mode B" section below.

## When NOT to use this skill

The three-file ceremony only pays off when the work is genuinely **multi-phase, multi-session, or cross-cutting** — that's what lets a plan survive context resets and hand off cleanly to a future you or another agent.

Do **not** spin up the three files for:

- Small component iterations / refactors (changing props, tweaking copy, adding defaults, adjusting styles, fixing one form validation)
- Single-file bug fixes / small feature touch-ups
- One-off cleanups, lint fixes, dependency bumps, copy edits
- Tasks that finish in a single session with no cross-system state to track

Test before bootstrapping: *if I tried to fill in `summary.md`'s "From → To" and "Definition of Done", would they read as trivial or contrived?* If yes, skip the skill and just do the task, with a one-line note to the user (e.g., "This is a small change; it doesn't need the phased-plan flow, so I'll just make the edit directly."). If the user pushes back ("no, please track this"), bootstrap normally — the user's intent wins.

Don't punish small tasks with heavyweight state files, **but** also don't let a real multi-session effort slip through as "just a small change." When the work spans files, services, sessions, or weeks — use the skill.

## First action, every time

Before doing anything else, check what's already on disk:

```bash
ls <repo>/.work-planner/ 2>/dev/null && cat <repo>/.work-planner/state.json 2>/dev/null
```

Decide:
- All three files exist with non-empty state → **Mode B (Resume)**.
- Directory missing, or files absent / blank → **Mode A (Bootstrap)** → read `references/bootstrap.md`.
- Partial / inconsistent (e.g., only `summary.md` exists) → ask the user before you create or overwrite anything.

Before starting any *new* plan when a plan already exists (`overall_status != "done"` means it's active): do **not** silently replace the three files. Show the user the current plan's goal, current step, next step, and completion rate, then ask for explicit confirmation. Without clear confirmation, default to resuming the existing plan. `references/bootstrap.md` covers the full guard + archive procedure.

Never overwrite an existing file without the user's go-ahead. State files are precious — they're what makes resumption work.

## Mode B: Resume an existing plan

Pick up where the last session left off, using only what's in `.work-planner/`. Don't trust your memory of past sessions — trust the files.

### Step 1: Read the state

```bash
cat <repo>/.work-planner/state.json
```

From it, extract:
- `current_step` and `next_step` — what to work on now
- the matching entry in `all_steps` — its `sub_steps`, `verify`, `depends_on`, per-step `notes`, and `refs`
- `decisions` — settled choices you must respect
- `failures` — anything to be careful of
- root `notes` — cross-step context

**Rehydrate context before implementing.** If the current step carries a `refs` list, open the matching rows in `summary.md`'s **Context & References** catalog and actually load what they point at — read the screenshot in `context/`, the pasted API doc, the spec excerpt, the referenced source file. This is the step that repairs the "cold session drifts from what the user showed" failure: a UI step without its mockup, or a client step without its API contract, will produce plausible-but-wrong work. When a step's output must match a reference (a screenshot, a contract, a spec), treat looking at that reference as part of the work, not optional.

Only re-read the rest of `plan.md` / `summary.md` if the state alone is ambiguous, or if you're about to change scope. They're long; don't load them wholesale on every turn — but the catalog rows for the current step's `refs` are cheap and always worth loading. If the plan is sharded (the step carries a `plan_file` field), load only that shard — `plan.md` is then just the index; don't load the other shards.

Before your **first write** to `state.json` in a session, read `references/state-spec.md` — it's the exact contract for statuses, timestamps, dependencies, and notes. Getting these wrong corrupts the audit trail that future sessions depend on.

### Step 2: Sanity-check before acting

- If the user's request and the state disagree (e.g., user says "do S5" but state says S5 is `done`), surface the conflict and ask — don't silently re-do work.
- If `overall_status` is already `done`, treat any new request as either a new phase to append or a separate follow-up — confirm which.
- **Check dependencies**: a step is only startable when every id in its `depends_on` is `done`. If a dependency is `blocked`, the step is not startable either — tell the user instead of working around it. Steps with no dependency relationship between them may be worked in any order (or in parallel by different sessions).

### Step 3: Execute one logical unit

Work the smallest meaningful unit — usually one sub-step. Match existing patterns in the target codebase. When in doubt about what "done" means, the step's `verify` field is the answer.

**If a step turns out much bigger than planned** (you're several sub-steps in and keep discovering more), don't grind on silently and don't nest a plan inside it. Stop, tell the user, and propose re-splitting it in place: replace the step with 2–4 new sibling steps (`S3` → `S3a`, `S3b`, `S3c`), carrying over completed work as already-`done` entries and wiring `depends_on` between the new steps. Update the plan file(s) and `state.json` together. The state stays flat — one `state.json`, one level of steps. If the re-split makes `plan.md` unwieldy, shard the plan instead of nesting (see `references/bootstrap.md`, "Sharding a large plan").

### Step 4: Update state immediately on completion

The moment a sub-step is verified or a step transitions status, update `state.json` **in the same turn**. Stale state is worse than no state — it makes the next session run the wrong thing.

Quick checklist (full rules in `references/state-spec.md`):

1. Sub-step done → flip its `status`; parent step `done` when all sub-steps are done.
2. Apply step-level `start_at` / `done_at` on transitions.
3. Update `current_step` / `next_step`, bump `completion_rate`, refresh `updated_at`.
4. Append discoveries to the right `notes` array (per-step by default).
5. On failure/block: append to `failures`, set the step `blocked`, stop and ask the user.

### New context arriving mid-session

The user often drops fresh material while work is underway — "here's the updated mockup", a new error screenshot, a link to the real API spec. Don't let it live only in the chat: the moment it's decision-relevant, persist a durable copy into `context/`, add a row to the **Context & References** catalog in `summary.md`, and add its id to the `refs` of every step it informs. If it contradicts something already recorded (a superseded design, a corrected contract), keep both — mark the old catalog row as superseded and note the change — so the audit trail shows why the target moved. The rule mirrors "files first, memory second": ephemeral chat context that isn't written down is context the next session loses.

## Working rules

1. **Files first, memory second.** If you "remember" a decision that's not in `.work-planner/`, write it down or ask the user to confirm.
2. **Match existing patterns in the target codebase.** Before introducing a new approach, grep for how similar things are already done.
3. **Update state on the same turn you finish work.**
4. **Don't escalate scope inside a step.** If you discover new work, add a sub-step or a new top-level step — don't quietly do it. If the step balloons, re-split it (see Mode B Step 3).
5. **When in doubt, read the source.** For ports / parity work, when behavior is ambiguous, read the original code rather than guess.
6. **Respect `.gitignore`.** Don't read or edit files under `.gitignore`.
7. **Surface assumptions.** When you act on an assumption recorded in `summary.md` and it turns out wrong, correct it there and note the impact — don't let the plan drift from stale assumptions.
8. **Persist context, don't reference the chat.** Screenshots, pasted docs, logs, and spec text that shaped the work go into `context/` and the catalog. A future session can't scroll back through this conversation — if it isn't on disk, it's gone.

## When the user asks something ambiguous

- "continue" / "next step" / "next" → Mode B; act on `next_step`.
- "start a new plan" / "start plan" / "help me break down this task" → Mode A; read `references/bootstrap.md`.
- "update the plan" → edit `plan.md` and, if step structure or dependencies changed, mirror into `state.json`.
- "what's the current progress" / "status" → summarize from `state.json` only; don't re-derive from the codebase unless asked.
- "audit the differences" → read `summary.md` for scope, then do a focused diff; record findings in the current step's `notes` or a `failures` entry.
- "make a note of this idea" → decide by scope: current step → per-step `notes`; cross-step → root `notes`.
