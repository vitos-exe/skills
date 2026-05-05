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
| [spotify-api](./skills/spotify-api/) | PKCE authentication, playlist management, track enrichment, and Liked Songs access. |
| [library-audit](./skills/library-audit/) | Audits one playlist per session: analyzes tracks, proposes a curatorial action (reframe/split/purge/merge), applies after confirmation, tracks progress in library-state.json. |
| [sort-inbox](./skills/sort-inbox/) | Routes new Liked Songs to the right playlist using the audited library fingerprint. Requires full audit complete. |
| [merge-scan](./skills/merge-scan/) | Scans all audited playlists for merge candidates based on thematic and vibe similarity. Fully offline discovery, applies confirmed merges via API. |
| [drift-sync](./skills/drift-sync/) | Detects drift between library-state.json and live Spotify playlists; reports missing copies, count changes, orphans, and deleted originals; syncs after confirmation. |

## Adding a New Skill

1. Create a folder under `skills/<skill-name>/`
2. Add a `SKILL.md` file with frontmatter:
   ```yaml
   ---
   name: skill-name
   description: When to trigger and what the skill does
   ---
   ```
3. Register in `.claude-plugin/marketplace.json` under the appropriate plugin's `skills` list
4. Update the Skills table in this README

See CLAUDE.md for development details.

## Structure

```
skills/                    # Skill implementations
.claude-plugin/
  marketplace.json        # Plugin registry (two plugins: my-skills, spotify)
CLAUDE.md                 # Development guide
```
