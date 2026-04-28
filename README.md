# vitos-exe/skills

Personal Claude Code skills repository. Custom skills for specialized workflows and automation.

## Quick Start

Install locally in Claude Code:

```bash
/plugin marketplace add vitos-exe/skills
```

Then use the skills by mentioning them or describing tasks they handle.

## Skills

### General

| Skill | Description |
|-------|-------------|
| [atomize](./skills/atomize/) | Breaks down code changes into atomic commits for review or commit modes. |

### Spotify

| Skill | Description |
|-------|-------------|
| [spotify-api](./skills/spotify-api/) | Comprehensive Spotify Web API reference: auth flows, playlists, search, playback, library, rate limiting, and Feb 2026 breaking changes. |
| [playlist-splitter](./skills/playlist-splitter/) | Splits a Spotify playlist into thematically coherent sub-playlists using a music analyst subagent. |

## Adding a New Skill

1. Create a folder under `skills/<skill-name>/`
2. Add a `SKILL.md` file with frontmatter:
   ```yaml
   ---
   name: skill-name
   description: When to trigger and what the skill does
   ---
   ```
3. Register in `.claude-plugin/plugin.json` under the appropriate plugin's `skills` list
4. Update the Skills table in this README

See CLAUDE.md for development details.

## Structure

```
skills/                    # Skill implementations
.claude-plugin/
  plugin.json             # Plugin registry (two plugins: my-skills, spotify)
CLAUDE.md                 # Development guide
```
