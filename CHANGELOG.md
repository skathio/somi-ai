# Changelog

All notable changes to `@skathio/somi-ai` are documented here.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) — versioning: [SemVer](https://semver.org/).

## [0.1.0] — 2026-05-21 — Initial release

First public release of SoMi AI.

### Added

#### Workflows and commands

- Three first-class workflows: **planning** (`/plan`), **coding** (`/code`), **reviewing** (`/review`), plus the full end-to-end pipeline (`/ship`).
- Supporting commands: `/plan-review`, `/security-review`, `/refactor`.
- Human-in-the-loop gates: every stage stops for explicit user approval before proceeding.

#### Agents

- **Core**: `planner`, `coder`, `reviewer`.
- **Support**: `security-reviewer`, `architecture-reviewer`, `test-strategist`, `refactorer`.

#### Planning — user-verified decisions

- The planner pauses inline on every architectural or design decision, presenting 2–4 concrete
  options with specific pros and cons (no vague phrasings), a recommendation, and two escape
  hatches: **Other** (user proposes a custom option) and **Discover** (guided narrowing questions
  to help the user choose by asking what favors or disadvantages each option).

#### Artifact model — `.somi/` directory

Every `/plan` invocation creates a work-item directory under `.somi/plans/<slug>/` containing:

- `context.md` — background, surrounding code, dependencies, constraints.
- `spec.md` — purpose, user story, requirements, core decision one-liners, DoD.
- `decisions.md` — ADR-style log: options, pros/cons, recommendation, discovery Q&A, reversibility. Decisions are never edited in place; stale ones are superseded by new entries.
- `progress.md` — single source of truth for status; phase table; in-flight work; open decisions.
- `diary.md` — append-only chronological narrative (newest first): plan changes, blockers, discoveries, review feedback.
- `phases/<NN>-*.md` — one file per phase, with iterations, acceptance criteria, test and observability changes, rollback steps.

Reviews are stored separately under `.somi/reviews/<slug>/`, one file per `/review` run.

#### Plan-change protocol

When implementation reveals the plan needs to change, the coder: updates `spec.md`, `decisions.md` (supersede, never edit), and `phases/` in place; appends a `diary.md` entry recording what changed and why; surfaces the change to the user before continuing. The plan never shows stale state.

#### Artifact templates

`CONTEXT.md.tmpl`, `SPEC.md.tmpl`, `DECISIONS.md.tmpl`, `PHASE.md.tmpl`, `PROGRESS.md.tmpl`,
`DIARY.md.tmpl`, `SOMI-README.md.tmpl`, `REVIEW.md.tmpl`, `ADR.md.tmpl`, `DOD.md.tmpl`.

#### Ruleset and skills

- Global ruleset (`rules/`) composing: priorities, SOLID, clean code, OWASP defenses, engineering practices, collaboration norms (including the user-verification protocol).
- On-demand skills: OWASP defense, SOLID principles, clean code, test strategy, API design, observability, threat modeling.

#### Deterministic guardrail hooks

Block dangerous shell commands, block secret writes, guard protected paths, lint changed files,
audit-log every tool call.

#### Distribution

- Claude Code plugin: marketplace manifest (`.claude-plugin/`) and npm package (`@skathio/somi-ai`).
- GitHub Copilot extension: `.copilot-extension/` manifest mirrors the Claude Code plugin.
- Validator workflow (`.github/workflows/validate.yml`): JSON, shellcheck, frontmatter checks.
- Release workflow (`.github/workflows/release.yml`).

#### Documentation and examples

Full documentation set: install, usage, workflows, agents, hooks, skills, rules, commands,
extending, versioning, governance, plugin, architecture.

Worked examples: feature plan (full six-artifact walkthrough), code review, end-to-end pipeline
transcript, and a sample consuming project showing the post-install layout.

[0.1.0]: https://github.com/skathio/somi-ai/releases/tag/v0.1.0
