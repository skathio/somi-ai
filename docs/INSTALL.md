# Installation

SoMi AI is distributed as a **Claude Code plugin** (via marketplace or npm) and as a **GitHub
Copilot extension** (via the Copilot plugin marketplace).

---

## Claude Code — plugin (marketplace)

The marketplace path is the recommended way to install SoMi AI into Claude Code.

```text
# 1. Register the SoMi AI marketplace source.
/plugin marketplace add https://github.com/skathio/somi-ai

# 2. Install the plugin.
/plugin install somi-ai@somi-ai
```

Once installed, `/plan`, `/code`, `/review`, `/ship`, and the supporting commands will appear in
Claude Code's `/` autocomplete.

### Updating

```text
/plugin update somi-ai
```

### Pinning a version

```text
/plugin pin somi-ai 0.1.0
```

Pinning is recommended for teams that want to adopt new versions deliberately rather than
automatically.

### Uninstalling

```text
/plugin uninstall somi-ai
```

---

## Claude Code — npm

If you prefer to manage the plugin through npm (e.g. to lock it in a `package.json`):

```bash
npm install -g @skathio/somi-ai
```

Then in Claude Code: `/plugin install somi-ai`.

---

## GitHub Copilot — extension marketplace

SoMi AI is also a GitHub Copilot extension, distributed through the same marketplace pattern as
the Claude Code plugin. The `.copilot-extension/` manifests mirror `.claude-plugin/` exactly —
both point at the same agent/command/skill/rules files.

### Install

```text
copilot plugin marketplace add https://github.com/skathio/somi-ai
copilot plugin install somi-ai@somi-ai
```

### Using the extension

Once installed, use `@somi-ai` in GitHub Copilot chat:

```text
@somi-ai /plan  Add per-team rate limiting to the public webhook endpoint
@somi-ai /code  rate-limiting-webhooks phase 1, iteration 1
@somi-ai /review  rate-limiting-webhooks
@somi-ai /ship  Full plan → code → review pipeline for: add audit logging
@somi-ai /security-review  rate-limiting-webhooks
@somi-ai /refactor  Untangle the payment service before patching
```

### Updating

```text
copilot plugin update somi-ai
```

---

## Prerequisites

| Distribution | Requirements |
|---|---|
| Claude Code plugin | Claude Code with `/plugin` support |
| npm | Node 18+, Claude Code with `/plugin` support |
| Copilot extension | GitHub Copilot subscription |

For lint hooks to run: the linter your project uses (`ruff`, `eslint`, `go vet`, etc.) must be on
`$PATH`. Missing linters are silently skipped rather than failing.

---

## Verifying the install

**Claude Code**: type `/` — you should see `/plan`, `/code`, `/review`, `/ship` in autocomplete.
Try `/plan list a trivial change` to confirm the planner agent loads.

**Copilot extension**: type `@somi-ai /plan test` in Copilot chat — SoMi AI should respond with
a structured plan.

If something is missing in Claude Code, check with `/plugin info somi-ai`.

---

## Troubleshooting

- **Hooks don't fire**: confirm `SOMI_ROOT` in `.claude/settings.json` resolves to the directory
  that contains `hooks/`. This is set automatically by the plugin installer.
- **`/plan` not visible after install**: reload the Claude Code window. Commands load at session
  start.
- **Copilot extension: `@somi-ai` not found**: confirm the extension is installed with
  `copilot plugin list` and that your Copilot subscription is active.

See also: [HOOKS.md](./HOOKS.md), [USAGE.md](./USAGE.md), [PLUGIN.md](./PLUGIN.md).
