---
name: sort-inbox
description: >
  Routes new Liked Songs to the right playlist using a fingerprint of the audited
  library. Trigger when the user wants to process their Liked Songs backlog and
  assign each track to the most fitting playlist. Requires the full library audit
  to be complete first.
---

# Sort Inbox

Fetches Liked Songs the user has not yet routed, matches each to the best-fit
playlist using the audited library fingerprint, and applies placements after
user confirmation.

**Prerequisite**: every playlist in `library-state.json` must have
`audit_status` of `"done"` or `"skipped"` -- none can be `"pending"`. If any
are still pending, stop and tell the user to finish the audit with
`spotify:library-audit` first. This is intentional: routing decisions are only
reliable once the library fingerprint is complete.

---

## Step 1: Credentials

Use the `spotify-api` skill for all auth and API calls. Required scopes:

```
playlist-read-private playlist-modify-public playlist-modify-private user-library-read
```

---

## Step 2: Load library-state.json

Read `library-state.json` from the current working directory.

- **Missing**: stop -- `spotify:library-audit` must be run first.
- **`playlists` array is empty**: stop -- the audit was never initialized. Run `spotify:library-audit` first.
- **Any playlist with `audit_status: "pending"`**: stop with a message listing
  the pending playlists and instructing the user to complete the audit.
- **All done/skipped**: proceed.

---

## Step 3: Fetch unprocessed Liked Songs

Fetch all Liked Songs via `GET /me/tracks` (paginated -- see spotify-api).

Filter out any track IDs already present in `routing_history` from
`library-state.json`. The remainder are the tracks to route this session.

If no new tracks are found, tell the user their inbox is empty and exit.

---

## Step 4: Enrich tracks

Using spotify-api patterns:

1. `GET /artists?ids=...` (batch, with 403 fallback)
2. Assemble enriched track list using `enrich_tracks` from spotify-api

Album enrichment is optional here -- artist genres are usually sufficient for routing.

---

## Step 5: Build library fingerprint

From `library-state.json`, collect all `done` playlists. For each:

```json
{
  "name": "Playlist Name",
  "description": "<description_written>",
  "sample_tracks": ["Track Name - Artist", "..."]
}
```

Use up to 10 sample tracks per playlist.

---

## Step 6: Match all tracks to playlists

Spawn a single subagent with this task:

```
You are a music librarian. For each track in the list below, pick the best-fit
playlist from the library. For tracks where two playlists are genuinely both
plausible, include both as options so the user can decide.

Library playlists:
{fingerprint as JSON}

Tracks to place:
{list of enriched tracks as JSON}

Return a JSON array, one entry per track, in the same order as the input:

[
  {
    "track_uri": "spotify:track:...",
    "track_name": "...",
    "artist": "...",
    "placement": "Playlist Name",
    "reason": "one sentence",
    "also_fits": "Other Playlist Name or null"
  }
]

Rules:
- "placement" must exactly match a playlist name from the library.
- "also_fits" is for genuine dual-placement candidates only -- when both playlists
  are a real fit, not just a loose association. Use null otherwise.
- Return only valid JSON. No prose before or after the array.
```

---

## Step 7: Present batch suggestions

Display all suggestions grouped by playlist:

```
Suggested placements:

{Playlist Name} ({N} tracks)
  - Track Name - Artist
  - Track Name - Artist

{Playlist Name} ({N} tracks)
  ...

Dual-placement candidates:
  - Track Name - Artist
      Option A: {Playlist A} - {reason}
      Option B: {also_fits}
```

Then ask:
> Does this look right? You can approve all, redirect specific tracks to different
> playlists, or skip any tracks.

---

## Step 8: Apply placements

For each approved track-playlist pair:
```
POST /playlists/{playlist_id}/items  { "uris": ["spotify:track:..."] }
```

Batch tracks going to the same playlist into a single call (up to 100 per call).

---

## Step 9: Update library-state.json

Append all processed track IDs to `routing_history`. Update `last_updated`.

Then update SESSION.md:

```markdown
# Session Log

## Last action
- Sorted {N} tracks from Liked Songs
- {summary of placements}

## Next step
- Run `spotify:sort-inbox` again when new Liked Songs accumulate
```
