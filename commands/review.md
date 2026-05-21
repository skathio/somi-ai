---
description: Strict, skeptical review of the current changes (or a specified diff/PR/plan). Severity-graded findings, evidence-driven, will reject weak solutions. Output lands under .somi/reviews/<slug>/.
argument-hint: <slug> | <diff range> | <PR #> | <file path> | "plan <slug>"
allowed-tools: Task, Read, Grep, Glob, Bash, Write, Edit, WebFetch
model: opus
---

# /review — Reviewing workflow

You are running the **reviewing workflow** of somi-ai.

The user's review target: **$ARGUMENTS** (empty = the current working-tree diff vs. the default
branch, scoped to the single in-progress work item if exactly one exists).

## What to do

### 1. Resolve the target

- **A work-item slug** (e.g. `rate-limiting-webhooks`) → review the latest iteration's diff against
  the spec/phases for that work item.
- **Empty / "working tree"** → `git diff` and `git status`. If exactly one work item has
  `status: in-progress` in `.somi/plans/`, scope the review to it.
- **A revision range** (e.g. `main..feature-x`) → use that range.
- **A PR number** (e.g. `#1234`) → fetch via `gh pr view --json` / `gh pr diff`; otherwise ask
  the user for the diff.
- **`plan <slug>`** → review the spec/decisions/phases, not code. (Same as `/plan-review <slug>`.)
- **A file path** → review the file (typically used for ADRs or design docs outside `.somi/`).

### 2. Read for intent first

Find the relevant work item's `spec.md`, the iteration's `phases/<NN>-*.md`, recent diary entries,
and commit messages. If you can't tell what the change is for, that's finding #1.

### 3. Brief the `reviewer` agent

Via the Task tool, pass:

- The target diff/file/plan.
- The work-item paths (`spec.md`, the iteration phase file, recent `diary.md` entries) so the
  reviewer can check for scope drift and plan-vs-code divergence.
- Hints about additional consultants:
  - Touches auth / crypto / input / file uploads / deserialization → **`security-reviewer`**.
  - Introduces new module / service / contract → **`architecture-reviewer`**.
  - Test shape problems → **`test-strategist`**.

### 4. Aggregate findings

A single severity-graded report (Blocker / Major / Minor / Nit, each with High / Medium / Low
confidence). Use [`templates/REVIEW.md.tmpl`](../templates/REVIEW.md.tmpl).

### 5. Write the review under `.somi/reviews/<slug>/`

Filename pattern: `<YYYY-MM-DD>-<phase>.<iter>-<verdict>.md` (e.g.,
`2026-05-21-iteration-1-2-request-changes.md`). If the review is not scoped to a work item (e.g.,
ad-hoc diff review), write to `.somi/reviews/_ad-hoc/<YYYY-MM-DD>-<slug>.md` instead.

### 6. Update work-item state (if scoped)

- In `progress.md`: append a line under "Recent activity" referencing the review file and verdict.
- If the review **affects the plan** (a Blocker reveals a missing decision, a Major points at a
  wrong assumption, etc.), append a diary entry with category `review-feedback` summarising what
  the review surfaced and link to the review file. The follow-up `/code` invocation will then
  apply plan changes per the plan-change protocol.

### 7. Summarise back

- **Verdict** (`approve` / `approve-with-comments` / `request-changes` / `reject`).
- **Counts** by severity.
- **Top 3 findings**, one line each, with severity.
- Pointer to the full review file under `.somi/reviews/<slug>/`.
- Next step:
  - `approve` / `approve-with-comments`: proceed to next iteration (`/code <slug>`) or merge.
  - `request-changes`: `/code <slug>` will pick up the review findings.
  - `reject`: discuss with the user — usually means re-plan, not re-code.

## Guardrails

- **Do not rubber-stamp.** If the diff is genuinely clean, say so with evidence (you read X, you
  traced Y).
- **Cite locations** with `path/to/file.ext:line-range` for every finding.
- **Grade honestly.** A long list of Nits is worse than one well-stated Blocker.
- **Reject when warranted.** Some changes shouldn't merge in any form — the right output is
  "reject" with the reason.
- **Plan-vs-code divergence is a finding**, even if the divergent code is technically fine.

## Quality bar

See [`agents/reviewer.md`](../agents/reviewer.md). Findings must include where, what,
why-it-matters, and a concrete suggested fix. Vague platitudes are not findings.
