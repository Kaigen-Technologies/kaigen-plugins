# Kaigen plugins for Claude Code

The `kaigen` plugin teaches a coding agent how to install the Kaigen 3D engine
CLI, license a machine, and create or repair a Kaigen project.

## Install

In Claude Code:

```
/plugin marketplace add Terminus-Technologies/kaigen-plugins
/plugin install kaigen@kaigen
```

Then just ask, e.g. *"set up Kaigen, my key is KGEN-XXXX-XXXX-XXXX"*.

Update later with `/plugin marketplace update`.

## Other agents

The same skill works anywhere that reads `SKILL.md` (Cursor, Codex, Copilot,
Windsurf, …):

```bash
npx skills add https://api.kaigen3d.com/skill.md -g
```

Or fetch it directly: <https://api.kaigen3d.com/skill.md>

## What's in here

```
.claude-plugin/marketplace.json          the catalog
plugins/kaigen/
  .claude-plugin/plugin.json             the plugin manifest (bump `version` to ship an update)
  skills/kaigen-setup/SKILL.md           the skill itself
```

`SKILL.md` is generated — it is published from the Kaigen backend repo by
`bun scripts/ops.ts skill ship`, which writes this copy and bumps the plugin
version. Edit it there, not here.

## Licensing

Using the engine requires a license key. Keys come from Kaigen —
<https://kaigen3d.com>. This repository contains documentation only.
