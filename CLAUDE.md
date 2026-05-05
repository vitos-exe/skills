# vitos-exe/skills

Personal Claude Code skills repo for [vitos-exe](https://github.com/vitos-exe).

## Structure

```
skills/
  <skill-name>/
    SKILL.md          # skill instructions + frontmatter
.claude-plugin/
  marketplace.json    # plugin registry (name, owner, skill list)
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
3. Register it in `.claude-plugin/marketplace.json` under the relevant plugin's `skills` array

## Installing locally

```bash
/plugin marketplace add vitos-exe/skills
```

## Skills

| Skill | Description |
|-------|-------------|
| [atomize](./skills/atomize/SKILL.md) | Breaks down code changes into atomic commits for review or commit modes |
| [spotify-api](./skills/spotify-api/SKILL.md) | PKCE auth, playlist CRUD, track enrichment (batch artist/album), and Liked Songs; all requests run inline via Bash |
| [library-audit](./skills/library-audit/SKILL.md) | Audits one playlist per session: analyzes, proposes reframe/split/purge/merge, applies after confirmation, tracks progress in library-state.json |
| [sort-inbox](./skills/sort-inbox/SKILL.md) | Routes new Liked Songs to the right playlist using the audited library fingerprint; requires full audit complete |
| [merge-scan](./skills/merge-scan/SKILL.md) | Scans all audited playlists for merge candidates based on thematic and vibe similarity; fully offline discovery, applies confirmed merges via API |
| [drift-sync](./skills/drift-sync/SKILL.md) | Detects drift between library-state.json and live Spotify playlists; reports missing copies, count changes, orphans, and deleted originals; syncs after confirmation |
