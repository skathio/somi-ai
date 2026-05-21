# Agents

SoMi AI ships seven subagents. The three **core** agents are the user-facing trio
(planner / coder / reviewer). The four **support** agents are invoked by the core agents (or
directly by the user) when the work clearly enters their domain.

| Agent                                                        | Tier     | When                                                                  |
|--------------------------------------------------------------|----------|-----------------------------------------------------------------------|
| [`planner`](../agents/planner.md)                            | core     | Non-trivial change; before any code is written                        |
| [`coder`](../agents/coder.md)                                | core     | Executing against an approved plan; small, well-scoped tasks          |
| [`reviewer`](../agents/reviewer.md)                          | core     | Before merge; whenever you want a skeptical second opinion            |
| [`security-reviewer`](../agents/security-reviewer.md)        | support  | Auth, crypto, secrets, input validation, deserialization, file uploads |
| [`architecture-reviewer`](../agents/architecture-reviewer.md)| support  | New module/service/contract; dependency direction change              |
| [`test-strategist`](../agents/test-strategist.md)            | support  | Test shape feels wrong; deciding unit vs. integration; flake debugging |
| [`refactorer`](../agents/refactorer.md)                      | support  | The next change needs untangling first; behavior-preserving structure  |

## How agents get invoked

Three paths:

1. **User invokes a command** (`/plan`, `/code`, `/review`) → command calls the corresponding
   core agent.
2. **A core agent escalates** during its work — e.g., coder hits auth code and asks whether to
   consult `security-reviewer`.
3. **User invokes a specialised command** (`/security-review`, `/plan-review`, `/refactor`) which
   directly targets a support agent.

SoMi AI prefers **explicit handoff** over silent specialisation. When a core agent thinks a
support agent should be consulted, it surfaces the recommendation; the human (or the
orchestrating command) decides.

## The core trio

### planner

Staff-engineer-grade planning. Produces the `.somi/plans/<slug>/` artifact set (context, spec,
decisions, progress, diary, phases). Pauses for **user verification** on every architectural or
design decision: presents 2–4 concrete options with explicit pros and cons (no vague phrasings),
recommends one, and offers `Other` (user-proposed option) plus `Discover` (guided narrowing
questions) as escape hatches.

- **Model**: `opus` (heavy judgment work).
- **Tools**: Read, Grep, Glob, WebFetch, Write, Edit, Bash.
- **Won't**: write code, silently pick architectural defaults.
- **Will**: stop and recommend re-scoping if the work is much larger than presented.

### coder

Elite implementation. Executes against the plan with senior-level design judgment. Updates
`progress.md`, the phase file, and `diary.md` as it works. Follows the **plan-change protocol**
when implementation reveals the plan needs changing: updates spec/decisions/phases in place,
appends a diary entry, surfaces to the user before continuing.

- **Model**: `opus`.
- **Tools**: Read, Edit, Write, Bash, Grep, Glob, WebFetch.
- **Won't**: silently widen scope; ship without running tests; bypass hooks; let the plan show
  stale state.
- **Will**: stop and trigger the plan-change protocol if the planned approach is producing bad
  code or hits an unforeseen constraint.

### reviewer

Strict, skeptical, evidence-driven. Reviews code, plans (the `.somi/plans/<slug>/` artifact set), or
architectural proposals. Checks plan-vs-code alignment: did the diff stay within scope, did
changes get captured in `decisions.md` and `diary.md`, is `progress.md` accurate.
Severity-graded findings, will reject weak solutions.

- **Model**: `opus`.
- **Tools**: Read, Grep, Glob, Bash, Write, Edit.
- **Won't**: rubber-stamp; bury Blockers under Nits; review the author instead of the code.
- **Will**: call in support agents when the change matches their territory; append a diary entry
  when a review surfaces a plan-level issue.

## The support quartet

### security-reviewer

OWASP-Top-10-lens audit. Trust-boundary-to-sink walks. Findings include **attack paths** in plain
language (preconditions, what gets executed, what the attacker gains), not just CVE-name dropping.

Invoke directly via `/security-review`, or via `/review` on a diff that touches sensitive
territory (the reviewer will recommend it).

### architecture-reviewer

Structural decisions — new module/service, dependency direction, public-contract introduction,
ADR review. Time horizon is years; reversibility is a first-class concern.

### test-strategist

Decides what to test, at what level, and how. Distinguishes risk-driven coverage from
coverage-worship. Identifies when the test shape is a *design* problem.

### refactorer

Surgical, behavior-preserving structure changes. Tests stay green at every step. No feature work
mixed in. Returns the codebase to a state where the next planned change is easy.

## Choosing model size

| Tier  | Default | When to override                                                                       |
|-------|---------|----------------------------------------------------------------------------------------|
| core  | `opus`  | Use `sonnet` for very simple `/code` tasks where the iteration is mechanical          |
| support | `opus` | Use `sonnet` for `architecture-reviewer` on small ADRs; `opus` for security and large reviews |

The model is set in each agent's frontmatter and can be overridden per-invocation via the Task
tool's `model` argument when calling from a command.

## Adding new agents

See [EXTENDING.md](./EXTENDING.md). The short version:

1. Add `agents/<name>.md` with proper frontmatter (`name`, `description`, `tools`, `model`).
2. Document it in this file with a one-row entry.
3. Open a PR — CI validates the frontmatter.

## Escalation matrix (which agent calls which)

```
planner          → coder        (handoff: .somi/plans/<slug>/ → iteration)
coder            → reviewer     (handoff: diff + summary + updated artifacts)
coder            → security-reviewer       (when sensitive territory)
coder            → architecture-reviewer   (when introducing structure)
coder            → test-strategist         (when test shape is unclear)
coder            → refactorer              (when next change needs untangling)
coder            → planner (re-plan)       (when plan-change protocol triggers a re-plan)
reviewer         → security-reviewer       (when reviewing sensitive diff)
reviewer         → architecture-reviewer   (when reviewing structural change)
reviewer         → coder (rework)          (Blocker/Major findings)
reviewer         → planner (re-plan)       (when the plan itself is wrong, not just the code)
refactorer       → test-strategist         (if coverage is too thin to refactor safely)
refactorer       → architecture-reviewer   (if the refactor is really structural)
test-strategist  → refactorer              (if untestable code is really untangleable)
test-strategist  → reviewer                (if a "flaky test" is actually a real bug)
```

## User verification protocol (planner-specific)

The planner has a **mandatory** verification protocol for any architectural or design decision
that shapes the spec:

1. **State the decision** in plain language.
2. **Offer 2–4 concrete options**, each with **specific pros and cons** (no vague phrasings —
   if you can't name concrete consequences, the option doesn't go on the list).
3. **Recommend** one with a one-or-two-sentence reason.
4. Offer **`Other`** (user proposes a different option) and **`Discover`** (agent asks narrowing
   questions to guide the choice) in every verification prompt.
5. Record the chosen option in `decisions.md` with `Verified with user: yes` and a one-liner in
   `spec.md` §5 (Core decisions).

Decisions changed mid-workflow are **never edited in place** — they're superseded by a new entry,
the old one stays marked `superseded by D<N>`, and a diary entry records the change.

See [`agents/planner.md`](../agents/planner.md) for the full protocol and examples.
