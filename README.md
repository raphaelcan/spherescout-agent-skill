# SphereScout Local Business Data Skill

An open Agent Skills-compatible workflow for finding, previewing, and exporting local business contacts through the SphereScout REST API.

The skill resolves business categories and locations, previews real match counts, and always asks for explicit approval before an export spends credits.

## Install

### Codex

Invoke `$skill-installer` and ask it to install:

`https://github.com/raphaelcan/spherescout-agent-skill/tree/v1.0.1/skills/spherescout-local-business-data`

Codex discovers the installed skill automatically. If it does not appear, restart Codex.

### Claude Code (CLI)

Copy `skills/spherescout-local-business-data/` into `.claude/skills/spherescout-local-business-data/` in your project (or `~/.claude/skills/spherescout-local-business-data/` to make it available in every project), then restart Claude Code. It will appear in the available-skills list.

### Claude.ai / Claude Desktop / Claude API

Upload the `skills/spherescout-local-business-data/` directory as a Skill (Settings → Capabilities → Skills, or the Skills API). Only `SKILL.md` is required; the `agents/openai.yaml` file in that directory is Codex/ChatGPT-only metadata and can be omitted.

### ChatGPT

This repository includes the plugin manifest and OpenAI interface metadata required for plugin distribution. Until the plugin is listed in the universal plugin directory, use the standalone skill in the ChatGPT desktop app's Skills interface or download the package for local installation.

### Skill package

The standards-compliant standalone skill is located at:

`skills/spherescout-local-business-data/`

### Other Agent Skills-compatible clients

Download the versioned release and install the `skills/spherescout-local-business-data` directory using your client's documented skill directory. The package follows the open Agent Skills specification.

### Manual fallback

Copy `skills/spherescout-local-business-data/SKILL.md` into a directory named `spherescout-local-business-data` in your agent's local skills folder.

## Authentication

Create a SphereScout API key at https://www.spherescout.io/dashboard and store it as `SPHERESCOUT_API_KEY`. Do not commit the key or paste it into `SKILL.md`.

## Safety boundary

Category search, location resolution, and previews do not consume export credits. The skill requires the agent to show the preview count and obtain explicit confirmation before exporting.

## License

MIT
