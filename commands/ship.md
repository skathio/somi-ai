---
description: Full plan → code → review pipeline against a single problem statement. Stops between stages for human approval. Operates over .somi/plans/<slug>/.
argument-hint: <problem statement>
allowed-tools: Task, Read, Edit, Write, Bash, Grep, Glob, WebFetch
model: opus
---

# /ship — End-to-end engineering pipeline

You are running the **full pipeline** of somi-ai: plan → code → review, with human-in-the-loop
gates between stages. All artifacts live under `.somi/plans/<slug>/`.

The user's problem statement: **$ARGUMENTS**

## Pipeline stages

### Stage 1 — Plan

1. Invoke the `/plan` flow (same logic as [`commands/plan.md`](./plan.md)) with `$ARGUMENTS`.
   This creates `.somi/plans/<slug>/` with `context.md`, `spec.md`, `decisions.md`, `progress.md`,
   `diary.md`, and `phases/`.
2. The planner **pauses inline** for any architectural/design decision and verifies with the user
   (with options/pros/cons, "Other", and "Discover" escape hatches). See `/plan` §6.
3. When planning is complete, `progress.md` status is `awaiting-approval`.
4. **STOP.** Present the summary. Explicitly ask:
   > "Plan ready under `.somi/plans/<slug>/`. Reply `approve` to proceed to Stage 2 (coding the first
   > iteration), `revise <notes>` to iterate on the plan, or `abort` to stop."

Do **not** proceed without an explicit `approve`. On approval, set `progress.md` status to
`in-progress`.

### Stage 2 — Code (one iteration at a time)

When the user approves, code **one iteration** at a time. Default to phase 1, iteration 1.

1. Invoke the `/code` flow with `<slug>` and the current iteration.
2. The coder follows the **plan-change protocol** if it discovers a constraint requiring spec/
   phase changes — updating files in place and appending a diary entry. See `/code` §5.
3. Produce the diff + tests. Update `progress.md` and `diary.md` per `/code` §7.
4. **STOP.** Present the summary. Explicitly ask:
   > "Iteration <N>.<M> implemented. Reply `review` to invoke the reviewer on this iteration,
   > `next` to proceed to the next iteration without a review (not recommended for non-trivial
   > iterations), or `stop` to pause the pipeline."

Default behavior is to review every iteration. The user can override.

### Stage 3 — Review

1. Invoke the `/review` flow against the current iteration's diff, scoped to `<slug>`.
2. Produce a review file under `.somi/reviews/<slug>/`.
3. If findings affect the plan, the review writes a diary entry (`/review` §6).
4. **STOP.** Present the verdict + top findings.
   - **`approve`**: ask "Reply `next` to proceed to the next iteration, or `stop` to pause."
   - **`approve-with-comments`**: same as approve, but list Minors the user may want to address.
   - **`request-changes`** or **`reject`**: loop back to Stage 2 with the findings as the brief.
     The `/code` agent applies fixes (and follows the plan-change protocol if the review surfaced
     a planning gap, not just a coding gap).

When all phases reach `done` in `progress.md`, the pipeline is complete. Set `progress.md` status
to `done` and append a final diary entry.

## Guardrails

- **Hard stops between stages.** No silent progression. The user must say yes.
- **One iteration per coding cycle.** Even if the plan has 5 iterations, you do not blast through
  them. Each gets its own code → review loop.
- **Re-plan on scope discovery.** If, during coding, the slice turns out to be wrong-sized, the
  plan-change protocol kicks in (spec/phase updates + diary entry) — pause and let the user see
  the revised plan before continuing.
- **Stop the pipeline on any Blocker finding** until it's resolved. Do not paper over.
- **Verification on architectural decisions** is mandatory during Stage 1; do not skip even when
  the user said "go fast".

## Why a pipeline command exists

Running `/plan`, `/code`, `/review` manually is correct and idiomatic. `/ship` exists for users
who want the full ceremony in one entrypoint with explicit gates — it does **not** skip review,
verification, or rubber-stamp anything; it just removes the boilerplate of typing the commands
separately.
