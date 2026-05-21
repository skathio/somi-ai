# Usage

Hands-on guide to running the three workflows. For each command, this doc shows: when to use it,
what to type, what to expect, and where the artifacts go.

## The fundamental loop

```
/plan <problem>            →  .somi/plans/<slug>/ created    →  user reviews + approves
   ↓                          (6 docs + phases/)
/code <slug>                →  diff + tests +          →  user inspects
       phase N, iteration M    progress/diary updates
   ↓
/review <slug>             →  .somi/reviews/<slug>/   →  findings addressed or accepted
                              <YYYY-MM-DD>-…md
   ↓
(loop iteration → next; or merge if done)
```

`/ship <problem>` runs the whole loop with hard gates between stages — same workflow, fewer
keystrokes.

---

## `/plan`

**When**: non-trivial change, multi-module work, anything touching security/auth/contracts, or any
request you can't restate in one sentence with confidence.

**Skip**: trivial single-file bug fix, doc-only changes, renames.

**Type**:
```text
/plan Add per-team rate limiting to the public webhook ingestion endpoint with audit logging and
      an emergency kill switch.
```

**Expect**:
- SoMi AI proposes a slug (e.g., `rate-limiting-webhooks`) and confirms with you.
- Reads the relevant code (Read/Grep/Glob).
- Drafts `context.md` (background, surroundings, constraints) and the spec skeleton.
- **Pauses on each architectural decision** — presents 2–4 concrete options with explicit pros and
  cons, recommends one, and offers `Other` (you describe a different option) or `Discover` (the
  agent asks narrowing questions to help you choose).
- Records each verified decision in `decisions.md` with the chosen option, rejected alternatives,
  rationale, and (if you used discovery mode) the narrowing Q&A.
- Writes phases under `phases/<NN>-<slug>.md`, sets `progress.md` to `awaiting-approval`, and
  appends the first `diary.md` entry.
- Summarises back: problem framing, phase count, top 3 risks, top 3 open questions, pointer to
  `.somi/plans/<slug>/`.
- **Stops.** Does not start coding.

**Then**:
- Read `.somi/plans/<slug>/spec.md` and `phases/01-*.md`.
- Edit any file directly if you want — they're your artifacts.
- Run `/plan-review <slug>` for a skeptical pass on the plan itself.
- When happy: `/code <slug>` (or `/code <slug> phase 1, iteration 1` for explicit targeting).

See [`examples/feature-plan-example.md`](../examples/feature-plan-example.md) for a worked output.

---

## `/code`

**When**: you have an approved plan; or, for trivial work, a self-contained task description.

**Type**:
```text
/code rate-limiting-webhooks                            # picks up next not-started iteration
/code rate-limiting-webhooks phase 1, iteration 1       # explicit target
/code Implement the in-memory RateLimiter described in phases/01-define-limiter.md
```

**Expect**:
- SoMi AI reads `spec.md`, the iteration's phase file, recent `diary.md`, and the surrounding
  code. Marks the iteration `in-progress` in `progress.md`.
- Edits or writes code, adds tests, runs them.
- If implementation reveals the plan needs to change (constraints, dead ends, false assumptions),
  it follows the **plan-change protocol**: updates spec/decisions/phases/progress in place,
  appends a `diary.md` entry, surfaces to you before continuing.
- Marks the iteration `done`, updates `progress.md`, appends a final diary entry.
- Summarises back: files changed, tests added, anything **not done**, plan changes (if any),
  tradeoffs, what to look at, next step (`/review <slug>`).

**Hook guardrails fire during this stage**: dangerous shell commands, secret writes, protected
paths are blocked deterministically. See [HOOKS.md](./HOOKS.md).

---

## `/review`

**When**: before merge; after each iteration; whenever you want a skeptical second opinion.

**Type**:
```text
/review rate-limiting-webhooks         # reviews the latest iteration's diff against the spec
/review                                # working-tree diff, scoped if exactly one work item is in-progress
/review main..feature-x                # reviews a revision range
/review #1234                          # reviews a GitHub PR (if gh available)
/review plan rate-limiting-webhooks    # reviews the spec/decisions/phases (alias for /plan-review)
```

**Expect**:
- Severity-graded findings: Blocker / Major / Minor / Nit, each with High / Medium / Low
  confidence.
- Plan-vs-code checks: did the diff stay within the iteration scope? Are decision changes
  captured?
- Written to `.somi/reviews/<slug>/<YYYY-MM-DD>-<phase>.<iter>-<verdict>.md`.
- A line in `progress.md` "Recent activity"; a diary entry if findings affect the plan.
- Summary: verdict, counts, top 3 findings.

If the diff touches auth/crypto/input-validation, the reviewer additionally consults the
`security-reviewer` agent. If it touches a new module/contract, it consults
`architecture-reviewer`.

See [`examples/code-review-example.md`](../examples/code-review-example.md) for a worked review.

---

## `/ship`

End-to-end pipeline: plan → code → review, with **hard human-in-the-loop gates** between stages.

**Type**:
```text
/ship Add a --dry-run flag to the migrate CLI that prints the SQL it would execute without
      applying.
```

**Expect**:
- Stage 1 (Plan): creates `.somi/plans/<slug>/`, pauses on every architectural decision for verification,
  finishes with `progress.md` status `awaiting-approval`. Stops, asks `approve` / `revise` /
  `abort`.
- Stage 2 (Code, first iteration): produces diff + tests, updates progress/diary, applies
  plan-change protocol if needed. Stops, asks `review` / `next` / `stop`.
- Stage 3 (Review): produces review file under `.somi/reviews/<slug>/`. Stops, asks based on
  verdict.
- Loops back to Stage 2 for the next iteration until done.

`/ship` does **not** skip review, verification, or rubber-stamp anything — it just removes the
boilerplate of typing each command separately.

See [`examples/full-pipeline-example.md`](../examples/full-pipeline-example.md) for a transcript.

---

## Specialised commands

### `/plan-review`

Reviews a plan before coding starts. Catches scoping errors, missing risks, weak decisions, and
bad architectural choices cheaply. Use it after `/plan` and before `/code`.

```text
/plan-review rate-limiting-webhooks
/plan-review docs/adr/0042-event-bus.md
```

### `/security-review`

Targeted security review. Walks trust boundaries to sinks and produces attack-path-grounded
findings.

```text
/security-review rate-limiting-webhooks   # scoped to a work item's latest iteration
/security-review main..feature-x          # range
```

Use this in addition to `/review` whenever the change touches auth / crypto / input / file uploads
/ deserialization / outbound HTTP from user input.

### `/refactor`

Surgical, behavior-preserving refactor of a named smell. Tests stay green; no feature work mixed
in.

```text
/refactor OrderService mixes pricing logic and persistence. Split pricing into a pure module and
          keep persistence behind a repository interface. Files: src/order/service.ts,
          src/order/repo.ts.
```

---

## What happens to the artifacts

| Artifact                              | Lives at                                             | Lifetime                                          |
|---------------------------------------|------------------------------------------------------|---------------------------------------------------|
| Work-item directory                   | `.somi/plans/<slug>/`                                      | Persists indefinitely; only you delete it         |
| `context.md`, `spec.md`, `decisions.md`, `progress.md`, `diary.md` | `.somi/plans/<slug>/`            | Same                                              |
| Phase files                           | `.somi/plans/<slug>/phases/<NN>-*.md`                      | Same                                              |
| Review files                          | `.somi/reviews/<slug>/<YYYY-MM-DD>-*.md`             | Same; one per review run                          |
| `audit.log`                           | `.claude/audit.log`                                  | Append-only across sessions                       |
| Diff                                  | git                                                  | As long as the branch / history is kept           |

All artifacts under `.somi/` should be committed to the repository. They're how the team and
future readers understand what was built and why.

## Multiple work items

Each `/plan` creates its own work item. Slugs come from the problem statement; you can pick a
different one when prompted. If you re-invoke `/plan` on the same problem, SoMi AI asks whether to
continue the existing work item (preserving diary), reset it, or branch into a new slug.

When invoking `/code` or `/review` without a slug, SoMi AI looks at `.somi/` for a single work
item with `status: in-progress` in its `progress.md` and uses it. If there are multiple, it asks.

## Plan changes during implementation

When the coder discovers something that requires changing the plan (not just the code), it
follows the **plan-change protocol**:

1. Stop the contested work.
2. Update `spec.md`, `phases/<NN>-*.md`, and `progress.md` in place.
3. In `decisions.md`, never edit an accepted entry — supersede it with a new one and mark the old
   one `superseded by D<N>`.
4. Append a `diary.md` entry (top of file, newest first) with category `plan-change` (or
   `decision-change`, `blocker`) explaining what was discovered and why the plan changed.
5. Surface to you with the revised plan before continuing.

The spec never shows stale state. The diary remembers what changed.

## Tips

- **Edit files in `.somi/plans/<slug>/` directly** between stages. They're your artifacts.
- **Use `/plan-review`** for anything you'd send to a human staff engineer for an architecture
  preview.
- **Re-run `/review`** after addressing findings. Verdicts can change — a Blocker fix sometimes
  reveals a new Major.
- **Commit `.somi/`** with the feature branch — the artifact set explains the work to future
  readers.
- **Inspect `audit.log`** if you're curious what tools SoMi AI touched during a session.
- **`diary.md` is the time machine** — when you come back to a work item in three months, read
  diary first to understand the journey.
