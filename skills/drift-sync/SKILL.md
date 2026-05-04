---
name: drift-sync
description: >
  Detects drift between library-state.json and actual Spotify playlists: missing
  agent copies, track count changes, orphaned [agent] playlists, and deleted
  originals. Reports all discrepancies grouped by type and applies confirmed sync
  actions. Trigger when the user wants to reconcile local tracking state with live
  Spotify library state.
---

# Drift Sync

Compares every entry in `library-state.json` against live Spotify data and fixes
discrepancies after user confirmation.

**Prerequisite**: `library-state.json` must exist. If missing, run
`spotify:library-audit` first.

---

## Step 1: Credentials

Use the `spotify-api` skill. Required scopes:

```
playlist-read-private playlist-read-collaborative playlist-modify-public playlist-modify-private
```

---

## Step 2: Load state

Read `library-state.json` from the current working directory.

Build these sets from the entries:

- **pending entries**: `audit_status == "pending"` -- `id` is the original playlist (not yet copied)
- **agent-copy entries**: `audit_status` in `["done", "skipped", "merged"]` -- `id` is the agent copy
- **original ids**: all non-null `original_id` values (originals left untouched during audit)
- **all tracked ids**: union of all `id` values + all non-null `original_id` values

Also note `reroute_playlist_id` at the root (may be null).

---

## Step 3: Fetch Spotify playlists

Fetch all playlists via `GET /me/playlists` (paginated). Build two maps:

- `spotify_by_id`: `{ id -> { name, track_count } }` for every playlist returned
- `agent_set`: set of IDs for every playlist whose `name` starts with `"[agent]"`

---

## Step 4: Detect drift

Check every category below. Collect results into a drift report.

### A. Missing agent copies

For each agent-copy entry (`done`/`skipped`/`merged`):
- If `id` not in `spotify_by_id`: agent copy was deleted from Spotify.
- Record: `{ type: "missing_copy", entry }`

### B. Track count drift

For each agent-copy entry where `id` IS in `spotify_by_id` and `track_count` is not null:
- If `spotify_by_id[id].track_count != entry.track_count`:
- Record: `{ type: "count_drift", entry, state_count: entry.track_count, spotify_count: actual }`

### C. Orphaned agent playlists

For each ID in `agent_set`:
- If ID not in `all tracked ids`:
- Record: `{ type: "orphan", id, name: spotify_by_id[id].name }`

### D. Missing originals (informational)

For each agent-copy entry with a non-null `original_id`:
- If `original_id` not in `spotify_by_id`: original was deleted.
- Record: `{ type: "missing_original", entry }` -- no sync action required, informational only.

### E. Pending originals gone

For each pending entry:
- If `id` not in `spotify_by_id`: original playlist was deleted before it could be audited.
- Record: `{ type: "pending_gone", entry }`

### F. Reroute playlist missing

If `reroute_playlist_id` is non-null and not in `spotify_by_id`:
- Record: `{ type: "reroute_missing", id: reroute_playlist_id }`

---

## Step 5: Report drift

If nothing was found, tell the user the library is in sync and exit.

Otherwise display a grouped summary:

```
Drift detected ({N} items):

[A] Missing agent copies ({N}):
  1. "{name}" (id: ...) — agent copy not on Spotify

[B] Track count changes ({N}):
  1. "{name}": state={X}, Spotify={Y}

[C] Orphaned [agent] playlists ({N}):
  1. "{name}" (id: ...) — not referenced in state

[D] Missing originals ({N}) [informational]:
  1. "{name}": original (id: ...) no longer on Spotify

[E] Pending playlists gone ({N}):
  1. "{name}" — original deleted before audit

[F] Reroute playlist missing:
  - id: ... — [agent] [reroute] no longer on Spotify
```

Then ask:
> Which actions would you like to apply?
> - [A] Mark missing agent copies as "lost" in state
> - [B] Update all stale track counts in state
> - [C] Delete orphaned [agent] playlists from Spotify
> - [E] Mark pending-gone entries as "lost" in state
> - [F] Re-create the reroute playlist
> - all — apply A + B + C + E (skips D, informational only)
>
> Or specify individual items by letter+number (e.g. "A1, C2").

---

## Step 6: Apply confirmed sync actions

### A: Mark missing copies as "lost"

For each confirmed missing-copy entry in state:
```json
{
  "audit_status": "lost",
  "notes": "agent copy deleted from Spotify -- detected by drift-sync on {ISO date}"
}
```

### B: Update track counts

For each count-drift entry in state:
- Set `track_count` to the live Spotify value.

### C: Delete orphaned playlists

For each confirmed orphan:
```
DELETE /playlists/{id}/followers
```
This unfollows the playlist. For agent-owned playlists with no other followers this effectively removes it.

### E: Mark pending-gone as "lost"

For each confirmed pending-gone entry in state:
```json
{
  "audit_status": "lost",
  "notes": "original playlist deleted before audit -- detected by drift-sync on {ISO date}"
}
```

### F: Re-create reroute playlist

```
POST /me/playlists  { "name": "[agent] [reroute]", "public": false, "description": "Holding area for purged tracks" }
```

Save the new ID to `reroute_playlist_id` in state.

---

## Step 7: Update library-state.json

Write all state changes. Update `last_updated` to the current ISO timestamp.

Update SESSION.md:

```markdown
# Session Log

## Last action
- Drift sync: {summary of what was found and fixed}

## Next step
- Run `spotify:drift-sync` again after further library changes
- Or continue with `spotify:library-audit` if pending playlists remain
```
