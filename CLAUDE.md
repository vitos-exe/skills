# vitos-exe/skills

Personal Claude Code skills repo for [vitos-exe](https://github.com/vitos-exe).

## Structure

```
skills/
  <skill-name>/
    SKILL.md          # skill instructions + frontmatter
.claude-plugin/
  plugin.json         # plugin registry (name, owner, skill list)
```

## Adding a skill

1. Create a directory under `skills/<skill-name>/`
2. Add a `SKILL.md` with required frontmatter:
   ```yaml
   ---
   name: skill-name
   description: When to trigger and what it does.
   ---
   ```
3. Register it in `.claude-plugin/plugin.json` under `plugins[0].skills`

## Installing locally

```bash
/plugin marketplace add vitos-exe/skills
```

## Skills

| Skill | Description |
|-------|-------------|
| [atomize](./skills/atomize/SKILL.md) | Breaks down code changes into atomic commits for review or commit modes |
| [spotify-api](./skills/spotify-api/SKILL.md) | Comprehensive Spotify Web API skill: auth flows, playlists, search, playback, library, rate limiting, Feb 2026 changes |
| [playlist-splitter](./skills/playlist-splitter/SKILL.md) | Splits a Spotify playlist into thematically coherent sub-playlists using a music analyst subagent |
