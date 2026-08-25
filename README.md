# SphereScout Local Business Data Skill

An open Agent Skills-compatible workflow for finding, previewing, and exporting local business contacts through the SphereScout REST API.

The skill resolves business categories and locations, previews real match counts, and always asks for explicit approval before an export spends credits.

## Install

### ChatGPT and Codex

Use the skill installer with this repository, or install the repository as a plugin when plugin installation is available in your client.

The standalone skill is located at:

`skills/spherescout-local-business-data/`

### Other Agent Skills-compatible clients

Download this repository and install the `skills/spherescout-local-business-data` directory using your client's documented skill directory.

### Manual fallback

Copy `skills/spherescout-local-business-data/SKILL.md` into a directory named `spherescout-local-business-data` in your agent's local skills folder.

## Authentication

Create a SphereScout API key at https://www.spherescout.io/dashboard and store it as `SPHERESCOUT_API_KEY`. Do not commit the key or paste it into `SKILL.md`.

## Safety boundary

Category search, location resolution, and previews do not consume export credits. The skill requires the agent to show the preview count and obtain explicit confirmation before exporting.

## License

MIT
