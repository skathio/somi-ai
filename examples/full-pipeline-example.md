# Example — `/ship` end-to-end pipeline

A condensed transcript of using `/ship` for a tiny feature: **add a `--dry-run` flag to the
`migrate` CLI**. Shows the human-in-the-loop gates between stages, the inline verification of
architectural decisions during planning, and how the `.somi/plans/<slug>/` artifact set gets populated.

---

## Turn 1 — User invokes `/ship`

```
> /ship Add a --dry-run flag to the migrate CLI that prints the SQL it would execute without
        applying any changes. Should work for both up and down migrations.
```

## Turn 2 — Claude (Stage 1: Plan — slug and scaffold)

> Proposed slug: `migrate-dry-run`. Creating `.somi/plans/migrate-dry-run/` …
>
> [scaffolds `context.md`, `spec.md`, `decisions.md`, `progress.md`, `diary.md`, `phases/`,
>  `reviews/`]
>
> Reading `cmd/migrate/main.go`, `internal/migrate/runner.go`, existing tests. Drafting
> `context.md` and the spec skeleton.

## Turn 3 — Claude (Stage 1: Plan — first verification)

> One architectural decision to verify before I write the phases:
>
> **Decision: shape of the dry-run option on `Runner`.**
>
> **Option A — `Options.DryRun bool`** — **Recommended**
> - Pros: minimal API surface; matches existing `Options.{Verbose, RetryCount}` style; one place
>   to read at the gate.
> - Cons: closes the door on future modes (`explain`, `verify`) without an additive change later.
>
> **Option B — `Options.Mode runner.Mode`** (typed enum: `apply` / `dryRun`)
> - Pros: leaves room for `explain` / `verify` modes by adding enum variants instead of new bools;
>   makes the gate exhaustive at compile time.
> - Cons: more code today for a one-mode addition; requires a default value when callers don't
>   supply one.
>
> **Other** — describe a different option.
> **Discover** — I'll ask narrowing questions.
>
> My recommendation: Option A. There's no concrete pull for `explain` / `verify` yet; we can
> migrate to an enum if and when we add a second alternative mode.

## Turn 4 — User

```
> A
```

## Turn 5 — Claude (Stage 1: Plan — finishing)

> Recorded as D1 in `decisions.md` (Verified with user: yes).
>
> Writing `phases/01-thread-flag.md`, `phases/02-tests-docs.md`. Updating `spec.md` §5 with the
> one-liner for D1. Initial `progress.md` written (status `awaiting-approval`). First diary entry
> appended.
>
> **Summary**: Two iterations under one phase. (1) Thread `--dry-run` through CLI and
> `Runner.Apply` using `Options.DryRun bool`. (2) Add tests + docs.
>
> **Risks**: (a) printing SQL with secrets — none of our migrations embed secrets, but worth a
> one-line redaction pass; (b) ordering — dry-run must still resolve dependencies in the same
> order as the real apply.
>
> **Open questions**: none.
>
> Plan ready under `.somi/plans/migrate-dry-run/`. Reply `approve` to proceed to Stage 2, `revise
> <notes>` to iterate on the plan, or `abort` to stop.

## Turn 6 — User

```
> approve
```

## Turn 7 — Claude (Stage 2: Code, Iteration 1.1)

> Marking phase 1, iteration 1 `in-progress` in `progress.md`. Reading the relevant files.
>
> [edits `cmd/migrate/main.go` to register the flag and pass it]
> [edits `internal/migrate/runner.go` to add `DryRun bool` to options and short-circuit `Exec` to
>  `fmt.Println`]
>
> Tests green. Marking iteration 1.1 `done`. Updated `progress.md`. Diary entry added.
>
> **Work item**: `migrate-dry-run` • iteration 1.1.
> **What changed**:
> - `cmd/migrate/main.go` — registered `--dry-run` flag; passed through to `Runner.Apply`.
> - `internal/migrate/runner.go` — added `Options.DryRun`; in dry-run mode, prints `BEGIN; <SQL>;
>   COMMIT;` for each migration in dependency order without opening a transaction.
> **Plan changes**: none.
> **Why**: matches phase 1 iteration 1 acceptance.
> **Not done**: tests + docs — planned for iteration 1.2.
> **What to look at**: the dependency-resolution path in `runner.go:142-158` — same code path as
> real apply (which is the whole point); if you re-order migrations, dry-run output reorders too.
> **Tradeoffs taken**: none material.
>
> Iteration 1.1 implemented. Reply `review` to invoke the reviewer on this iteration, `next` to
> proceed to the next iteration without a review, or `stop`.

## Turn 8 — User

```
> review
```

## Turn 9 — Claude (Stage 3: Review)

> [reads the diff, `spec.md`, `phases/01-…`, recent diary entries]
>
> **Verdict**: `approve-with-comments`.
> **Counts**: 0 Blockers, 0 Majors, 1 Minor, 1 Nit.
>
> **Top findings**:
> - **[Minor / Medium]** Dry-run prints `BEGIN/COMMIT` per migration, but real apply uses a single
>   transaction per `up` batch. Output suggests semantics that don't quite match real apply.
>   Suggest matching the real shape or noting the divergence in the printed header.
> - **[Nit / Low]** None material on naming or structure.
>
> Plan-vs-code: no drift; iteration scope matched `phases/01-thread-flag.md` exactly.
>
> Review file at `.somi/reviews/migrate-dry-run/2026-05-21-iteration-1-1-approve-with-comments.md`.
> Recent activity logged in `progress.md`. No diary entry needed — finding is a code concern, not
> a plan concern.
>
> Reply `next` to proceed to iteration 1.2 (tests + docs), `revise` to address the Minor first,
> or `stop`.

## Turn 10 — User

```
> revise — address the Minor by matching the real transaction shape in dry-run output.
```

## Turn 11 — Claude (Stage 2 again, addressing review feedback)

> Adjusting iteration 1.1 per review feedback.
>
> [edits `internal/migrate/runner.go` so dry-run emits one `BEGIN` at the start of a batch and one
>  `COMMIT` at the end, matching real apply semantics]
>
> Done. Re-running review …
>
> **Verdict**: `approve` (clean). Review file updated.
>
> Reply `next` to proceed to iteration 1.2.

## Turn 12 — User

```
> next
```

## Turn 13 — Claude (Stage 2, Iteration 1.2)

> [writes table tests for dry-run output; updates `docs/migrate.md`]
>
> ... [final review on iteration 1.2 — clean approve] ...
>
> All iterations complete. Setting `progress.md` status to `done`. Final diary entry appended.
> Work item: `migrate-dry-run`.
>
> Artifacts at `.somi/plans/migrate-dry-run/` — full audit trail of the work. Ready for PR.

---

## What the example demonstrates

- **Slug + scaffold up front** — every work item gets its own `.somi/plans/<slug>/` directory; the
  six-file artifact set is created from templates before drafting starts.
- **Inline verification on architectural decisions** — the agent surfaces options with concrete
  pros and cons and offers `Other` / `Discover` escape hatches. The user decides; the choice is
  recorded in `decisions.md`.
- **Hard gates** between stages — Claude never silently progresses.
- **One iteration per coding cycle** — even though the plan had two iterations, each one got its
  own code → review loop.
- **Progress + diary updates** — `progress.md` reflects status; `diary.md` captures notable
  events. If the coder had to change the plan mid-implementation, a diary entry with category
  `plan-change` would have recorded what changed and why.
- **Reviews are scoped to the work item** — they land under `reviews/` and update `progress.md`.
  A plan-affecting finding would have appended a `review-feedback` diary entry.
- **Artifacts persist** — `.somi/plans/migrate-dry-run/` is the durable record of the work, auditable
  after the session ends.
