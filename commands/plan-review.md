---
description: Review a work item's plan (spec + decisions + phases) before coding starts. Catches scoping errors, missing risks, unstated assumptions, and bad architectural choices early.
argument-hint: <slug> | <path to plan file>
allowed-tools: Task, Read, Grep, Glob, Write, Edit
model: opus
---

# /plan-review — Review a plan before coding

You are running a **plan-level review** of somi-ai.

The target: **$ARGUMENTS** (a work-item slug, or a path to a single plan file like an ADR).

## What to do

### 1. Resolve the target

- **A work-item slug** → read `.somi/plans/<slug>/spec.md`, `decisions.md`, `phases/*.md`,
  `context.md`. The plan is the spec + decisions + phases together.
- **A path to a single file** (e.g., an ADR at `docs/adr/0042-event-bus.md`) → review that file.
- **Empty** → if exactly one work item has `status: awaiting-approval` or `planning` in
  `.somi/plans/`, use it. Otherwise ask the user.

### 2. Read the plan in full

Don't skim. The point of plan review is to catch errors that are cheaper to fix here than after
code is written.

### 3. Brief the `reviewer` agent

Pass:

- All the plan files (or the ADR).
- The `context.md` (essential — many plan errors are context errors).
- An explicit request for a **plan-level review** (not a code review).
- Hints about additional consultants:
  - Plan introduces a new module/service/contract or changes dependency direction →
    **`architecture-reviewer`** as well.

### 4. Aggregate findings

Plan-level severity grading:

- **Blocker** — plan as written will produce a wrong or unsafe outcome.
- **Major** — plan will produce excessive rework, missed risk, or design lock-in.
- **Minor** — plan is roughly right but a section is weak.
- **Nit** — taste/preference, no rework needed.

Use [`templates/REVIEW.md.tmpl`](../templates/REVIEW.md.tmpl) with plan-level framing.

### 5. Write the review

Filename pattern: `<YYYY-MM-DD>-plan-review-<verdict>.md` under `.somi/reviews/<slug>/`. For
standalone ADR reviews, write next to the ADR or under `.somi/reviews/_ad-hoc/`.

### 6. Update work-item state (if scoped)

- `progress.md`: line under "Recent activity" referencing the plan review and verdict.
- If findings require plan changes, append a diary entry with category `review-feedback`. The
  user (or next `/plan` revision) applies the changes.

### 7. Summarise back

- **Verdict** (`approve` / `approve-with-comments` / `request-changes` / `reject`).
- **Counts** by severity.
- **Top 3 findings** with one line each.
- Pointer to the review file.

## What to look for in a plan

- **Restatement mismatch** — does `spec.md` §1 match the user's actual problem?
- **Missing non-goals** — vague non-goals breed scope creep.
- **Unstated assumptions** — beliefs the plan depends on that aren't called out in `context.md`.
- **Decision quality** — are entries in `decisions.md` real choices (with rejected alternatives
  carrying reasons), or do they read like the agent picked without thinking?
- **Phase shapes** — coherent, reviewable, reversible? Or pseudo-phases ("implement / test /
  deploy")?
- **Iteration sizes** — is each ~1 PR? If not, split.
- **Risk realism** — specific failure modes with specific mitigations, or generic platitudes?
- **Security blind spots** — does `spec.md` §8 acknowledge auth/crypto/PII surfaces? Does it
  gate `security-reviewer` invocation in the right phase?
- **Test strategy** — risk-driven, or coverage-worship?
- **Rollout & rollback** — flag plan, deployment, rollback steps?
- **Definition of Done** — measurable, not vibes-based?
- **Open questions** — does `progress.md` acknowledge what the human still needs to decide?
- **Verification gaps** — are there decisions in `decisions.md` that *should* have been verified
  with the user but weren't?

## Guardrails

- Plans are cheaper to fix than code. Be **more skeptical** during plan review than code review —
  push back on weak choices.
- **Reject** when the plan needs to be re-thought, don't just request changes.
- A plan with a beautifully filled-in template but no actual choices is **worse** than a plan with
  blunt decisions and visible tradeoffs. Reward substance over completeness.
