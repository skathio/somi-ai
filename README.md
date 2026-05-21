# SoMi AI  `v0.1.0`

> An opinionated, reusable multi-agent engineering workflow system for Claude Code and GitHub Copilot.

This is the **initial release** of SoMi AI — see [CHANGELOG.md](CHANGELOG.md) for the full feature set.

SoMi AI gives engineering teams a shared, version-controlled "operating system" for working with Claude:
three first-class workflows — **plan → code → review** — backed by specialised subagents, deterministic guardrail hooks,
composable skills, and a global ruleset that enforces SOLID, clean code, and OWASP defenses.

It is designed to be:

- **Reusable** across many repositories and teams
- **Installable** as a Claude Code plugin (via marketplace or npm) or as a GitHub Copilot chat extension
- **Opinionated but extensible** — strong defaults, clean overrides
- **Deterministic where possible** (hooks) and **judgment-heavy where needed** (agents)

---

## The three workflows

| Command       | Workflow  | Agent       | Purpose                                                                                  |
|---------------|-----------|-------------|------------------------------------------------------------------------------------------|
| `/plan`       | Planning  | `planner`   | Staff-engineer-grade plan: phases, risks, slices, DoD, test & rollout strategy           |
| `/code`       | Coding    | `coder`     | Execute against an approved plan with senior-level design judgment                       |
| `/review`     | Reviewing | `reviewer`  | Strict, skeptical review of code / plans / architecture with severity-graded findings    |
| `/ship`       | Pipeline  | all three   | Full plan → code → review pipeline against a single problem statement                    |

Supporting agents (used by handoff): `security-reviewer`, `architecture-reviewer`, `test-strategist`, `refactorer`.

---

## Install

### Option 1 — Claude Code plugin (marketplace)

```text
# Add the SoMi AI marketplace and install the plugin:
/plugin marketplace add https://github.com/skathio/somi-ai
/plugin install somi-ai@somi-ai
```

Updates flow through `/plugin update somi-ai`.

### Option 2 — npm (Claude Code)

```bash
npm install -g @skathio/somi-ai
```

Then in Claude Code: `/plugin install somi-ai`.

### Option 3 — GitHub Copilot (extension marketplace)

SoMi AI is also a GitHub Copilot extension, installable the same way as the Claude Code plugin:

```text
copilot plugin marketplace add https://github.com/skathio/somi-ai
copilot plugin install somi-ai@somi-ai
```

Once installed, use `@somi-ai` in GitHub Copilot chat:

```text
@somi-ai /plan  Add per-team rate limiting to the public webhook endpoint
@somi-ai /code  rate-limiting-webhooks phase 1, iteration 1
@somi-ai /review  rate-limiting-webhooks
```

---

## What's in the box

```
.claude-plugin/   Plugin + marketplace manifests (Claude Code plugin distribution)
agents/           Subagent definitions (planner, coder, reviewer, + support)
commands/         Slash-command entrypoints (/plan, /code, /review, /ship, ...)
skills/           On-demand expert knowledge packs (OWASP, SOLID, test strategy, ...)
rules/            Global ruleset composed into CLAUDE.md
hooks/            Deterministic guardrails (block dangerous bash, secret writes, ...)
templates/        Artifact templates (CONTEXT, SPEC, DECISIONS, PHASE, PROGRESS, DIARY, REVIEW, ADR, DOD)
.copilot-extension/ Copilot extension + marketplace manifests (mirrors .claude-plugin/)
examples/         Worked examples + a minimal consuming project
docs/             Full documentation
```

When you use SoMi AI in a project, workflows write their artifacts into a `.somi/` directory
at the project root. Plans live under `.somi/plans/<slug>/` (context, spec, decisions, progress,
diary, phases); reviews live under `.somi/reviews/<slug>/` — separate directories, no clutter.
See [`docs/WORKFLOWS.md`](docs/WORKFLOWS.md) for the full layout.

---

## Quick start (after install)

```text
> /plan  Add per-team rate limiting to the public webhook ingestion endpoint
        with audit logging and an emergency kill switch.

# Claude proposes a slug ("rate-limiting-webhooks"), reads the repo, drafts context.md,
# then pauses inline on each architectural decision — presenting options with concrete
# pros/cons, plus "Other" and "Discover" escape hatches. You decide. Verified decisions
# land in decisions.md; the spec, phases, progress, and diary fill in. Review the
# artifacts at .somi/plans/rate-limiting-webhooks/, edit if needed, approve.

> /code  rate-limiting-webhooks phase 1, iteration 1

# Claude implements with senior-engineer judgment, writes tests, updates docs, and
# keeps the plan in sync — if implementation reveals the plan needs to change, it
# updates spec/decisions/phases in place and appends a diary entry.

> /review  rate-limiting-webhooks

# Claude returns severity-graded findings (written under reviews/), rejects weak
# solutions, flags plan-vs-code divergence.
```

For the all-in-one pipeline: `/ship <problem statement>`.

---

## Why a shared OS, not per-project setups

- **Consistency** — every repo gets the same review bar, the same security posture, the same plan shape.
- **Upgrade once** — update the plugin; every project benefits.
- **Override locally** — projects keep their own `CLAUDE.md` and `rules/99-overrides.md`; SoMi AI never silently overrides them.
- **Auditable** — hooks log denied actions; reviewers can see what the system blocked vs. what humans approved.

---

## Documentation

- [Installation](docs/INSTALL.md) — Claude Code plugin, npm, or Copilot extension
- [Usage](docs/USAGE.md) — running each workflow with examples
- [Workflows](docs/WORKFLOWS.md) — plan / code / review semantics and handoffs
- [Agents](docs/AGENTS.md) — full agent catalogue, escalation rules
- [Hooks](docs/HOOKS.md) — guardrails and how to add your own
- [Skills](docs/SKILLS.md) — on-demand expertise packs
- [Rules](docs/RULES.md) — global ruleset philosophy and conflict resolution
- [Commands](docs/COMMANDS.md) — slash-command reference
- [Extending](docs/EXTENDING.md) — adding workflows, agents, skills
- [Versioning](docs/VERSIONING.md) — SemVer policy, breaking-change rules
- [Governance](docs/GOVERNANCE.md) — how teams adopt updates safely
- [Plugin distribution](docs/PLUGIN.md) — marketplace and VS Code setup
- [Architecture](docs/architecture.md) — how the pieces fit together

---

## Versioning

SoMi AI follows [Semantic Versioning](https://semver.org/). The `VERSION` file is the source of truth.
See [docs/VERSIONING.md](docs/VERSIONING.md) for the breaking-change policy and migration guide template.

Current version: **0.1.0** — initial release.

---

## License

MIT — see [LICENSE](LICENSE).
