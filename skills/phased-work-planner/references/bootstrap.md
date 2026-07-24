# Bootstrap a new phased plan

Read this file only when `.work-planner/` is missing / empty, or when the user explicitly asks to create a new plan.

## 1. Guard and archive any existing plan

If `.work-planner/state.json` exists, inspect `overall_status` before writing anything.

- **Unfinished (`!= "done"`)**: show the current goal, `current_step`, `next_step`, and `completion_rate`. Ask whether to resume or explicitly replace it. Without clear confirmation, resume.
- **Completed (`== "done"`)**: ask whether this is a follow-up phase (append) or a separate plan (archive first).

When replacement is confirmed, archive the **entire plan bundle** together before creating new ones. This includes the raw context directory; never archive only the three markdown/JSON files:

```
.work-planner/
├── archive/<task-name>/
│   ├── summary.md
│   ├── plan.md
│   ├── state.json
│   ├── plan-*.md                 # if sharded
│   └── context/                   # screenshots, pasted docs, logs, etc.
├── summary.md
├── plan.md
├── state.json
└── context/
```

Use `mv` on each existing plan artifact (`summary.md`, `plan.md`, `state.json`, every `plan-*.md`, and `context/`) so contents remain byte-for-byte intact. Never overwrite an archive. Choose `<task-name>` from the archived plan's goal: lowercase kebab-case, ≤40 chars; if occupied, append `-2`, `-3`, etc. Confirm the name before moving. If `context/` is absent in a legacy plan, record that the archive has no captured raw context rather than inventing files.

## 2. Inspect before interviewing

Before asking the user broad questions, inspect enough of the repo to avoid making them describe facts the code can answer. Maintain a strict boundary: discoverable facts come from the environment; decisions that require preference, risk acceptance, or product judgment belong to the user.

- top-level layout and relevant packages/services
- similar implementations and conventions
- build/test/deploy entry points
- obvious integration boundaries and configuration

Then build a small **uncertainty map**. Look especially for areas that are easy to overlook yet expensive to discover late:

- external APIs/services and undocumented contracts
- shared database/schema, event formats, queues, caches, or storage
- authentication, authorization, identity propagation, and secrets
- backward compatibility, old/new coexistence, rollback, and data ownership
- runtime/deployment topology, feature flags, observability, and operational handoff
- cross-team dependencies, approvals, deadlines, or unavailable environments
- behavior whose source of truth is outside the inspected repo

Do not turn this into exhaustive threat modeling. Prioritize uncertainties using two dimensions: **blast radius if wrong** and **cost of discovering late**.

## 3. Interview one decision at a time

Ask only what the conversation and repository cannot answer. Establish the basics:

1. Current state and desired end state.
2. Scope and explicit non-goals.
3. Constraints / coexistence / rollout.
4. Concrete, verifiable definition of done.

Treat the interview as a dependency-ordered decision tree, not a questionnaire. Start with the unresolved decision that has the largest downstream impact, then follow the branch created by the user's answer before moving to an independent decision.

For each decision:

1. Ask **one question at a time** and wait for the answer. Do not batch several decisions into one message.
2. Explain briefly why the decision changes the plan.
3. Provide a recommended answer based on the repository evidence, stated constraints, and common engineering practice. Include the main trade-off so the user can accept or override it deliberately.
4. Resolve dependent decisions before moving sideways to an unrelated branch.
5. Restate the settled choice when the answer is ambiguous; do not silently infer agreement.

For example:

> Will the old and new services share the same auth token? This determines whether they can run in parallel during rollout and whether testing needs cross-system session compatibility.
>
> **Recommendation:** share the token validation contract during coexistence, but keep token issuance owned by one service. This minimizes session disruption without creating two competing authorities.

Be persistent where a decision has a large blast radius or would be expensive to reverse late. Stop probing when the remaining uncertainty is low-impact, reversible, or discoverable cheaply during execution. The goal is shared understanding, not a fixed question count or an exhaustive interrogation.

Avoid generic checklists such as "any other risks?". If an uncertainty matters but does not warrant blocking the interview, state a provisional assumption and invite correction. Record unresolved items rather than silently guessing.

## 4. Capture raw context into `context/`

Do this **before** distilling the summary, while the source material is still in front of you. Scan the triggering conversation for reference material the plan will depend on but the prose summary can't fully carry:

- screenshots / images (target UI, error dialogs, diagrams)
- pasted API docs, request/response samples, schemas, config
- error logs / stack traces
- spec excerpts, requirement paragraphs, acceptance criteria
- reference URLs (design systems, external docs, tickets)
- example files or data the user handed you

For each item, decide **persist vs. reference**:

- **Persist** anything ephemeral — screenshots and pasted text exist only in this chat. Save a durable copy under `.work-planner/context/` with a descriptive kebab-case name that says what it is: `login-mockup.png`, `orders-api-contract.md`, `npe-stacktrace.txt`. Save images as image files; save pasted text/docs as `.md`/`.txt`. Don't paraphrase into the filename's content — store it verbatim.
- **Reference** anything already durable and in-repo — a source file to match, a committed spec. Point at its path; no need to copy.
- **URLs**: record the URL in the catalog. If the page may change or vanish and it's central, also save a fetched copy into `context/`.

Then build the **Context & References catalog** (a table in `summary.md`, below). Give each item a stable id `C1`, `C2`, … — these ids are what `plan.md` steps and `state.json` `refs` point at. If the conversation genuinely carried no reference material, write "None" and move on; don't fabricate entries.

## 5. Write `.work-planner/summary.md`

Keep it concise (normally under ~150 lines):

```markdown
# Plan Summary

## Goal
<from-state → to-state, and why>

## Scope
- In: ...
- Out: ...

## Constraints / Coexistence
<shared systems, rollout, deadlines, hard constraints>

## Definition of Done
<concrete, verifiable signals>

## Context & References
| id | Source | Location | What it's for |
|---|---|---|---|
| C1 | Login screen mockup (screenshot) | context/login-mockup.png | pixel/layout target for S2 |
| C2 | Orders API (pasted doc) | context/orders-api-contract.md | request/response contract for S3 |
| C3 | Existing auth module | src/auth/token.ts | pattern to match for S1 |
| C4 | Design system tokens | https://... | colors/spacing, S2 |

## Assumptions and Open Questions
| Item | Status | Why it matters | Resolution point |
|---|---|---|---|
| ... | assumed / open / confirmed | ... | before S2 / during S1.2 / ... |

## Key Decisions (locked)
| Decision | Choice | Why |
|---|---|---|
| ... | ... | ... |
```

The `Location` column is either a `context/<file>` path (persisted), a repo-relative source path (referenced), or a URL. Do not manufacture an open-question section when there are no meaningful uncertainties. Each open question should have a resolution point so it does not remain invisible until late execution.

## 6. Write `.work-planner/plan.md`

The blueprint must be usable by a future session without re-deriving the work:

```markdown
# Plan

## Target architecture
<directory layout, module boundaries, key components>

## Dependency graph
S1 ──┬──> S2 ──> S4
     └──> S3 ──┘

## Phases / Steps
### S1 — <name>
- Goal: <success looks like>
- Depends on: none
- Refs: C3 — match the existing auth pattern
- Resolves: <open questions / assumptions this step validates, if any>
- Sub-steps:
  - S1.1 ...
  - S1.2 ...
- Verify: <how we know S1 is done>

### S2 — <name>
- Depends on: S1
- Refs: C1, C4 — mockup + design tokens are the visual target
...
```

Attach a `Refs:` line to any step whose correct execution depends on a catalog item (a UI step needs its mockup; a client step needs its API contract). A step with no reference material simply omits the line. Naming the *why* next to each ref tells a future session what to look for, not just that something exists.

Dependency rules:

- Model dependencies between first-level steps with `depends_on`. Do not use list order as a hidden dependency.
- Keep sub-step dependencies implicit in their listed order unless non-linear execution materially matters.
- Prefer a shallow directed acyclic graph. Reject cycles.
- If every step is a simple linear chain, keep the graph compact (`S1 → S2 → S3`) rather than drawing ceremony.
- Steps with no dependency path between them are independent and may run concurrently.
- Do not create nested phased plans. If a step later grows too large, re-split it into sibling steps in this same plan.

## Sharding a large plan

A single `plan.md` works for most efforts. But when the blueprint itself becomes huge (rough signal: heading past ~400 lines, or several step groups each need their own architecture/layout detail), a monolithic file punishes every future read — resume sessions load pages of irrelevant blueprint to find one step.

In that case shard by step group, and turn `plan.md` into a thin index:

```
.work-planner/
├── summary.md
├── plan.md              # index: goal recap, dependency graph, shard table
├── plan-auth.md         # S1–S3 in full detail
├── plan-data-layer.md   # S4–S6 in full detail
├── plan-rollout.md      # S7–S8 in full detail
└── state.json
```

Rules:

- `plan.md` (the index) keeps: the overall dependency graph across **all** steps, a one-line summary per step, and a table of shards (`plan-<slug>.md` → which step ids it covers). It should stay short enough to read in one glance.
- Each shard holds the full detail (goal, sub-steps, verify, architecture notes) for its steps only. A step lives in exactly one shard.
- Shard names: `plan-<slug>.md`, lowercase kebab-case, named after the step group's theme — not `plan-1.md` / `plan-2.md`.
- Link state to shards: every first-level step in `state.json` gains a `plan_file` field (e.g., `"plan_file": "plan-auth.md"`). On resume, load only the shard the current step points to.
- **If there is only one `plan.md`, do not add `plan_file` anywhere** — the field's absence means "the plan is unsharded". Don't pre-shard a plan speculatively; start monolithic and shard only when it actually grows too large (that later split follows the same rules: update the index, move step details into shards, add `plan_file` to every step in the same write).
- The dependency graph lives in the index only — never duplicated per shard, so it can't fall out of sync.

## 7. Initialize `.work-planner/state.json`

Read `state-spec.md` for the full contract. Initial shape:

```json
{
  "version": "1.3.0",
  "created_at": "YYYY-MM-DD HH:mm:ss",
  "updated_at": "YYYY-MM-DD HH:mm:ss",
  "current_step": "S1",
  "next_step": "<short prose>",
  "overall_status": "in_progress",
  "completion_rate": "0%",
  "all_steps": [
    {
      "id": "S1",
      "name": "<matches plan.md>",
      "status": "pending",
      "depends_on": [],
      "refs": ["C3"],
      "notes": [],
      "sub_steps": [
        { "id": "S1.1", "name": "...", "status": "pending" }
      ],
      "verify": "<one-line criterion>"
    }
  ],
  "failures": [],
  "decisions": {},
  "notes": []
}
```

Set each step's `refs` to the catalog ids from its plan `Refs:` line (omit or use `[]` when the step needs none). These are the ids a resuming session resolves against the Context & References catalog before implementing the step.

Choose `current_step` as a pending step whose `depends_on` is empty. If multiple roots exist, choose the one that reduces the most uncertainty or unlocks the most downstream work; mention other runnable roots in `next_step` when useful.

## 8. Confirm shared understanding before execution

Show the user the goal, assumptions/open questions, settled decisions, step list, dependency graph, and the Context & References catalog. Ask for corrections and obtain explicit confirmation that the plan reflects the shared understanding before starting S1. Do not write implementation code during bootstrap.
