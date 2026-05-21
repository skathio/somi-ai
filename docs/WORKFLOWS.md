# Workflows

SoMi AI organises Claude's behavior into three first-class workflows. Each has a clean handoff to
the next. Each produces durable artifacts inside `.somi/plans/<slug>/`. Each can be invoked alone or as
part of `/ship`.

## The three workflows

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   PLANNING      │ ───▶ │     CODING      │ ───▶ │   REVIEWING     │
│   /plan         │      │     /code       │      │   /review       │
│   agent:planner │      │   agent:coder   │      │ agent:reviewer  │
│  → .somi/plans/<slug>/│      │   → diff+tests  │      │ → reviews/...md │
│   (6 docs +     │      │  + updates spec/│      │ + updates diary │
│   phases/)      │      │  diary on change│      │ if plan affected│
└─────────────────┘      └─────────────────┘      └─────────────────┘
        ▲                                                  │
        │                                                  │
        └──────────────  re-plan if blocker  ──────────────┘
```

## Where artifacts live

Plans and reviews live in separate subdirectories to avoid cluttering work-item directories with
review output:

```
.somi/
├── README.md
├── plans/
│   └── <slug>/                         ← one directory per work item
│       ├── context.md                  ← background, surrounding code, constraints
│       ├── spec.md                     ← purpose, requirements, decisions, user story, DoD
│       ├── decisions.md                ← ADR-style log of architectural choices
│       ├── progress.md                 ← single source of truth for status
│       ├── diary.md                    ← chronological narrative of changes and discoveries
│       └── phases/
│           ├── 01-<slug>.md            ← one file per phase, iterations inside
│           └── …
└── reviews/
    └── <slug>/                         ← reviews scoped to a work item
        ├── 2026-05-21-iter-1-1.md
        └── …
```

The `.somi/` directory holds **both current and past work** — work items are not auto-archived.
Status lives in `progress.md`, not in the directory location. Only humans delete from `.somi/`.

## Planning

**Purpose**: produce deep implementation plans before any code is written.

**Agent**: [`planner`](../agents/planner.md).

**Input**: a problem statement from the user.

**Output**: the six-file artifact set under `.somi/plans/<slug>/` plus phase files. At minimum the
artifacts together capture: problem framing, goals/non-goals, assumptions, unknowns, architecture
sketch, decisions considered (with rejected alternatives carrying reasons), sequenced phases,
PR-sized iteration slices, test strategy, security considerations, observability plan, rollout
& rollback, risk register, definition of done, and open questions.

**User verification on decisions** — every architectural or design decision goes through:
1. Present the decision in plain language.
2. Offer 2–4 concrete options, each with specific (non-vague) pros and cons.
3. Recommend, with reason.
4. Offer **Other** (user proposes a custom option) and **Discover** (guided narrowing questions)
   as escape hatches.
5. Record the choice in `decisions.md` with `Verified with user: yes`.

**Quality bar**: a different engineer should be able to read `spec.md` + `phases/01-*.md` and
start coding **without asking another question**. Decisions are visible and arguable. Risks are
concrete failure modes with concrete mitigations.

**Stops the workflow**: never starts coding. The human must approve.

**Handoff to coding**: explicit. Code references the work-item slug and the phase/iteration being
executed.

## Coding

**Purpose**: implement against an approved plan with senior-level design judgment, keeping the
plan in sync with reality.

**Agent**: [`coder`](../agents/coder.md).

**Input**: a work-item slug + iteration reference (e.g., `phase 1, iteration 1`), or a
self-contained trivial task.

**Output**: a coherent diff + tests + updated docs (when behavior changes) + updates to
`progress.md`, `phases/<NN>-*.md`, and `diary.md` + a summary identifying what changed, what was
not done, what to look at first.

**Plan-change protocol** — if implementation reveals the plan needs to change:
1. Stop the contested work.
2. Update `spec.md`, `decisions.md` (supersede entries; never edit in place), `phases/<NN>-*.md`,
   `progress.md` to reflect the new truth.
3. Append a diary entry with category `plan-change` / `decision-change` / `blocker`.
4. Surface to the user before continuing.

The plan never shows stale state. The diary remembers what changed.

**Quality bar**: tests pass locally (the agent ran them), naming/structure match surrounding code,
no scope drift, no silent compromises, no leftover debug, plan kept in sync if changed.

**Re-plans on scope discovery**: see plan-change protocol above. Coder doesn't silently widen
scope; it surfaces and the user decides.

**Handoff to reviewing**: explicit. The reviewer reads the spec, the diff, recent diary entries,
and the summary.

## Reviewing

**Purpose**: strict, skeptical, evidence-driven review of code, plans, or architectural proposals.

**Agent**: [`reviewer`](../agents/reviewer.md). Calls in
[`security-reviewer`](../agents/security-reviewer.md),
[`architecture-reviewer`](../agents/architecture-reviewer.md), or
[`test-strategist`](../agents/test-strategist.md) when the change matches their territory.

**Input**: a work-item slug (most common), or a diff (working tree, range, PR), a plan, an ADR, or
a file.

**Output**: a review file at `.somi/reviews/<slug>/<YYYY-MM-DD>-<phase>.<iter>-<verdict>.md` with
severity-graded findings (Blocker / Major / Minor / Nit), each with a location, what's wrong, why
it matters, and a suggested fix. Plus a line in `progress.md` "Recent activity" and a diary entry
if findings affect the plan.

**Plan-vs-code checks** (SoMi-specific) — does the diff stay within the iteration scope? Did plan
changes get captured in `decisions.md` and `diary.md`? Is `progress.md` accurate?

**Quality bar**: no rubber-stamping. If the diff is clean, the reviewer says so with evidence
(read X, traced Y). Findings cite specific `file:line` locations. Reject when warranted.

**Handoff back to coding (rework)**:
- **Blocker** — must fix before merge.
- **Major** — should fix; merging without resolution requires explicit human sign-off.
- **Minor** — nice to fix; can be follow-up.
- **Nit** — style/taste, no obligation.

When findings point at the *plan* (not just the code), the reviewer says so and the next `/code`
run applies the plan-change protocol.

---

## Why these three (and not four, or five)

The split tracks the **three reasons engineering work is hard**:
- **Planning** — knowing what to build and in what order.
- **Coding** — executing without introducing new problems.
- **Reviewing** — catching what the executor missed.

These three exist in every engineering team's day; SoMi AI makes them explicit and gives each one
a specialised agent with a clear quality bar.

Support agents (`security-reviewer`, `architecture-reviewer`, `test-strategist`, `refactorer`) are
*facets* of these three, invoked when the work clearly engages their domain. They aren't separate
workflows because they don't have separate problem-shapes; they're depth-on-demand.

## When workflows compose

- **Plan → Code → Review** is the normal sequence.
- **Plan → Plan-review → Code → Review** when the plan is high-stakes or high-ambiguity.
- **Code → Review → Code (rework) → Review** when the first review surfaces findings.
- **Plan → Code → Review → Plan-change protocol → Code** when review reveals the plan was wrong,
  not just the code. Spec/decisions/phases get updated in place; diary entry captures why.
- **Refactor (standalone)** when the next planned change requires untangling first; refactor is
  its own mini-cycle that returns the codebase to a state where the planned change is easy.

## The `/ship` shortcut

`/ship <problem>` runs the full pipeline with hard gates between stages. It's identical to running
`/plan`, then `/code`, then `/review` manually — just with the orchestration baked in. Use
whichever feels natural; the underlying agents, artifacts, and quality bars are the same.

## What SoMi AI workflows are *not*

- **Not a substitute for human judgment.** The human approves between stages and decides on every
  architectural choice (the agent recommends; the user picks).
- **Not a one-shot.** Each stage is iterative; review feedback flows back into coding; coding can
  flow back into planning.
- **Not silent.** Every stage produces durable artifacts you can read, edit, and reject. The diary
  records when and why the plan shifts.
- **Not destructive of history.** Past work items remain in `.somi/`; superseded decisions stay
  in `decisions.md`; old diary entries are never rewritten.
