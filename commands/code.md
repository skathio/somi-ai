---
description: Execute against an approved plan in .somi/plans/<slug>/ with senior-level design judgment. Updates progress.md and diary.md as it goes. Specify which work item and phase/iteration to run.
argument-hint: <slug> [phase N, iteration M] | <free-form task>
allowed-tools: Task, Read, Edit, Write, Bash, Grep, Glob, WebFetch
model: opus
---

# /code — Coding workflow

You are running the **coding workflow** of somi-ai.

The user's request: **$ARGUMENTS**

## What to do

### 1. Resolve the work item

- If `$ARGUMENTS` starts with a slug matching a directory in `.somi/plans/`, that's the work item.
- If `$ARGUMENTS` is bare phase/iteration syntax (e.g., `phase 1, iteration 2`), look at `.somi/plans/`
  for a single work item with `status: in-progress` in its `progress.md` — use that. If multiple
  are in-progress, ask the user which one.
- If `$ARGUMENTS` is a free-form task description and **no work item exists** or applies:
  - If the work is **trivial and self-contained** (one file, one purpose), proceed without a work
    item. No artifacts to update.
  - If the work is **non-trivial**, stop and recommend `/plan <problem>` first.

### 2. Locate the iteration

Read `.somi/plans/<slug>/spec.md`, `progress.md`, and the relevant `phases/<NN>-*.md`. Find the
iteration the user named (or, if unspecified, the first iteration with status `not-started` after
all earlier ones are `done`).

If the iteration is `blocked`, surface why (from `progress.md` and `diary.md`) and ask whether to
unblock first, switch iterations, or proceed despite the block.

### 3. Brief the `coder` agent

Via the Task tool, pass:

- The work-item slug and `.somi/plans/<slug>/` paths.
- The specific phase + iteration to execute.
- The text of that iteration's section (scope, acceptance, files, tests).
- A reminder of the **plan-change protocol** (see §5 below).
- Any context from the current conversation.

The coder ([`agents/coder.md`](../agents/coder.md)) handles the implementation.

### 4. Mark iteration in-progress

Before the coder starts, update `progress.md`:

- Iteration status: `not-started` → `in-progress`.
- Set the **"Currently in flight"** section to this iteration.
- Update `Last activity` line.

### 5. Plan-change protocol (mid-coding adjustments)

If during implementation the coder discovers a constraint or blocker that requires changing the
plan itself (not just the code):

1. **Stop coding** on the contentious part.
2. **Update the affected files in place**:
   - `spec.md` — update Core decisions, requirements, or DoD as needed.
   - `decisions.md` — supersede the old entry; add a new one. Never edit a decided ADR in place.
   - `phases/<NN>-*.md` — update scope, acceptance, files, or split into more iterations.
   - `progress.md` — reflect the new state.
3. **Append a diary entry** to `diary.md` (top of file) with:
   - Category: `plan-change` (or `blocker` / `decision-change` as fits).
   - One paragraph: what was discovered, what changed in the plan, why.
   - Links to the updated docs.
4. **Surface to the user** before continuing: "Plan adjusted. Spec/decisions/phases updated.
   Diary entry at top. Proceed with revised plan, or want to revisit?"

The plan does **not** stay stale. The diary remembers what changed.

### 6. Verify

Run the iteration's tests yourself, or inspect the diff. Do not declare done on the coder's word
alone.

### 7. Mark iteration done + update progress

When the iteration is complete and tests are green:

- In `phases/<NN>-*.md`: set the iteration status to `done`.
- In `progress.md`:
  - Update the phase row (iterations done / total).
  - Move this iteration out of "Currently in flight".
  - Update `Last activity` line.
  - If all iterations in the phase are now `done`, set the phase status to `done` and check
    whether the next phase is ready to start.
- Append a short diary entry: category `note`, one line summarising what was implemented.

### 8. Summarise back

Return with:

- The work-item slug + iteration that was implemented.
- Files changed (one line each).
- Tests added/changed.
- Anything **not done**, with reason.
- Any **plan changes** made during this iteration, with diary link.
- Anything that crossed into security/architecture territory — and whether `security-reviewer` /
  `architecture-reviewer` should be invoked before `/review`.
- A specific next step: `/review <slug>` to validate before merging.

## Guardrails

- **No drive-by refactors.** Out-of-scope improvements go in `progress.md` follow-ups.
- **No widening scope without confirmation.** If the iteration is wrong-sized, follow §5
  (plan-change protocol) — don't silently expand.
- **No silent compromises.** Disabled tests, suppressed lints, removed assertions → name them in
  the summary and in a diary entry.
- **Tests must be runnable and green** before declaring done. If they can't run in this
  environment, say so.
- **Never edit `decisions.md` entries in place** once accepted. Supersede with a new entry.

## Quality bar

See [`agents/coder.md`](../agents/coder.md). Matched the iteration, tests green, no leftover debug,
no scope drift, surfaced any tradeoffs, plan kept in sync if it changed, diary entry made for any
non-trivial discovery.
