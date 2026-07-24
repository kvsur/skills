# `state.json` contract

Read before initializing or writing `.work-planner/state.json`.

## Schema and statuses

The current schema version is `1.3.0`. Preserve exact field names. Version `1.2.0` plans remain readable; when a session writes one after this update, migrate it to `1.3.0` and add `refs: []` to each first-level step if no context catalog exists.

Allowed step and sub-step statuses:

- `pending` — not started
- `in_progress` — partially done
- `done` — verified complete
- `blocked` — needs user or external input

`overall_status` uses `in_progress`, `blocked`, or `done`. Do not invent `paused`, `skipped`, or `wip`; use the closest allowed value and explain in `notes`.

Each first-level step has:

```json
{
  "id": "S2",
  "name": "...",
  "status": "pending",
  "depends_on": ["S1"],
  "refs": ["C1", "C4"],
  "notes": [],
  "sub_steps": [
    { "id": "S2.1", "name": "...", "status": "pending" }
  ],
  "verify": "..."
}
```

`depends_on` contains first-level step ids only. Every referenced id must exist. Dependencies must form a directed acyclic graph.

**`refs`**: first-level step ids of Context & References catalog items (`C1`, `C2`, …) whose material this step depends on — a mockup, an API contract, a spec excerpt, a source file to match. Every id must exist as a row in `summary.md`'s catalog. A resuming session loads these before implementing the step, so the work matches what the user actually provided rather than a lossy summary. Use `[]` (or omit) when the step needs no reference material. When new context arrives mid-work, add its catalog id to the `refs` of every step it informs in the same write that updates the catalog.

**`plan_file`** (optional): present only when the plan is sharded into multiple `plan-<slug>.md` files. Names the shard that holds this step's full detail. When the plan is a single `plan.md`, omit the field entirely — never write `"plan_file": "plan.md"` or `null`. If any step has `plan_file`, every step must; sharding is all-or-nothing.

## Dependency transitions

A step may move `pending → in_progress` only when every step in `depends_on` is `done`.

- A blocked dependency blocks all downstream steps from starting, but downstream statuses remain `pending` unless they have an independent reason to be `blocked`.
- When choosing `current_step`, prefer a runnable step that either resolves a high-impact open question or unlocks the most downstream work.
- If several independent steps are actively being worked by different sessions, `current_step` points to the step owned by the current session; note parallel work in root `notes` rather than changing the schema.
- When dependencies change, update `plan.md` and `state.json` together and check the graph for cycles.

## Timestamps

All timestamps use current local time exactly as `YYYY-MM-DD HH:mm:ss`: no timezone suffix, milliseconds, or `T` separator.

- `created_at`: set once at bootstrap; never modify.
- `updated_at`: refresh on every state write.
- `start_at`: first-level steps only; add on the first `pending → in_progress` transition.
- `done_at`: first-level steps only; add when the step transitions to `done`.

`start_at` and `done_at` are absent by default — never write empty strings, `null`, or placeholders. Sub-steps do not carry timestamps.

If a `done` step reopens because of a regression, keep its original timestamps and append a note explaining the reopen. A `blocked` step keeps `start_at` and has no `done_at`.

## Completion and pointers

When a sub-step is verified, immediately mark it `done`. Mark the parent step `done` only when all sub-steps are done **and** its `verify` criterion passes.

Calculate `completion_rate` as completed sub-steps divided by total sub-steps, rounded to a useful whole percentage. If a step has no sub-steps, count the step itself as one unit.

After every transition:

1. Pick `current_step` from the runnable steps.
2. Write `next_step` as concrete short prose, including the sub-step id where possible.
3. Refresh `completion_rate` and `updated_at`.

When everything finishes: set `overall_status: "done"`, `completion_rate: "100%"`, ensure the final step has `done_at`, and leave `current_step` pointing at the last completed step for audit readability.

If no unfinished step is runnable because of a genuine external block, set `overall_status: "blocked"` and describe the unblock condition in `next_step`.

## Failures

On failure or external block, append:

```json
{ "step": "S2.1", "reason": "<specific reason and unblock condition>", "at": "YYYY-MM-DD HH:mm:ss" }
```

Set the affected first-level step to `blocked` and stop for required input. Never erase previous failure entries after recovery; append a note documenting the resolution.

## Notes discipline

Notes are arrays of plain strings and append-only.

- **Per-step `notes`**: default location for discoveries about that step or its sub-steps. Prefix the sub-step id in the string when relevant.
- **Root `notes`**: only for cross-step / plan-wide observations.

Do not duplicate a note at both levels, rewrite/reorder old notes, or collapse arrays into strings. Add a timestamp prefix when it helps future readers.

## Re-splitting an oversized step

Do not create a nested plan. After user confirmation, replace the oversized step with 2–4 sibling steps in both `plan.md` and `state.json`:

1. Preserve completed work as `done` sub-steps/steps and retain its notes.
2. Give new ids derived from the original (`S3a`, `S3b`, `S3c`) to keep the audit trail obvious.
3. Redirect all dependencies that previously targeted `S3` to the new terminal replacement step(s).
4. Add `depends_on` among replacements to represent actual ordering.
5. Recompute `current_step`, `next_step`, and `completion_rate`.
6. Append a root note explaining why and when `S3` was re-split.

Never silently discard the original step's history. If keeping a retired `S3` entry is clearer for audit, retain it as `done` only when its completed portion genuinely passed verification; otherwise move its history into the replacement steps' notes and remove it consistently from both files.
