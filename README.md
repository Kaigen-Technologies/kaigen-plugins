# Kaigen plugins for Claude Code

The `kaigen` plugin teaches a coding agent how to install the Kaigen 3D engine
CLI, license a machine, and create or repair a Kaigen project.

## Install

**Claude Code**

```
/plugin marketplace add Kaigen-Technologies/kaigen-plugins
/plugin install kaigen@kaigen
```

Update later with `/plugin marketplace update`.

**Codex**

```
codex plugin marketplace add Kaigen-Technologies/kaigen-plugins
```

Then install `kaigen` from the plugin directory. Restart Codex if it does not
appear.

**Anything else** (Cursor, Copilot, Windsurf, …)

```bash
npx skills add https://api.kaigen3d.com/skill.md -g
```

Or read it directly: <https://api.kaigen3d.com/skill.md>

Once installed, just ask — e.g. *"set up Kaigen, my key is KGEN-XXXX-XXXX-XXXX"*.
You will need a license key; keys come from Kaigen.

## What's in here

One skill, three packaging conventions over the same directory:

```
.claude-plugin/marketplace.json          Claude Code catalog
.agents/plugins/marketplace.json         Codex catalog
plugins/kaigen/
  .claude-plugin/plugin.json             Claude Code manifest
  .codex-plugin/plugin.json              Codex manifest
  plugin.json                            Agent Plugins 1.0 manifest (Cursor, Copilot, VS Code, Kiro)
  skills/kaigen-setup/SKILL.md           the skill — shared by all of them
```

`skills/<name>/SKILL.md` is the discovery location every one of those specs
agrees on, so the skill is stored once. The manifests carry a `version`, which
is the update signal — `ops skill ship` bumps all of them together.

`SKILL.md` is generated — it is published from the Kaigen backend repo by
`bun scripts/ops.ts skill ship`, which writes this copy and bumps the plugin
version. Edit it there, not here.

## Licensing

Using the engine requires a license key. Keys come from Kaigen —
<https://kaigen3d.com>. This repository contains documentation only.
