---
name: library-audit
description: >
  Audits a Spotify playlist one at a time: analyzes tracks using a music analyst
  subagent, proposes a curatorial action (reframe/split/purge/merge), applies it
  after user confirmation, and tracks progress in library-state.json. Trigger when
  the user wants to work through and clean up their Spotify library.
---

# Library Audit

Works through a Spotify library playlist by playlist. Each session picks the next
unaudited playlist, analyzes it, applies a confirmed action, and saves state so any
future session can continue where this one left off.

---

## Step 1: Credentials

Use the `spotify-api` skill to handle all auth and API calls. Required scopes:

```
playlist-read-private playlist-read-collaborative playlist-modify-public playlist-modify-private
```

---

## Step 2: Load or initialize library-state.json

Read `library-state.json` from the current working directory.

**If missing**: fetch all playlists via `GET /me/playlists` (paginated -- see spotify-api).
Populate `library-state.json` with every playlist as `pending`:

```json
{
  "version": 1,
  "phase": "audit",
  "last_updated": null,
  "playlists": [
    {
      "id": "...",
      "name": "...",
      "audit_status": "pending",
      "mess_score": null,
      "action_taken": null,
      "description_written": null,
      "sample_tracks": [],
      "notes": null
    }
  ],
  "routing_history": []
}
```

Tell the user how many playlists were found and that the audit is starting.

**If exists with pending entries**: pick the next target by `mess_score` descending (null scores = pick first in list order). The user can override by naming a specific playlist. If the user asks to skip the current playlist, set its `audit_status` to `"skipped"` and pick the next pending one.

`mess_score` is a manual priority field (0-10). It is always `null` when first populated and the audit proceeds in list order. The user can set it by hand in `library-state.json` to prioritize known problem playlists.

**If exists with no pending entries**: congratulate the user -- the audit is complete. Suggest switching to Phase 2 by changing `"phase"` to `"routing"` in the state file and using `spotify:sort-inbox`.

---

## Step 3: Fetch and enrich the target playlist

Using spotify-api patterns:

1. `GET /playlists/{playlist_id}` -- get playlist name and metadata
2. `GET /playlists/{playlist_id}/items` (paginated) -- collect all tracks
3. `GET /artists?ids=...` (batch, with 403 fallback) -- enrich with artist data
4. `GET /albums?ids=...` (batch, with 403 fallback) -- enrich with album data
5. Assemble enriched track list using the `enrich_tracks` pattern from spotify-api

---

## Step 4: Build library fingerprint

From `library-state.json`, collect all playlists with `audit_status: "done"`.
For each, build a fingerprint entry:

```json
{
  "name": "Playlist Name",
  "description": "<the description_written field>",
  "sample_tracks": ["Track Name - Artist", "..."]
}
```

Use up to 10 sample tracks per playlist, formatted as "Track Name - Artist Name".

If no done playlists exist yet, pass an empty fingerprint -- the analyst will still work,
it just cannot recommend `merge`.

---

## Step 5: Spawn the analyst subagent

Read `ANALYST.md` from the same directory as this skill file. Use its full contents as
the system prompt for a subagent invoked via the Agent tool.

Pass the following as the subagent's task:

```
Source playlist: "{playlist_name}"
Total tracks: {N}

Library fingerprint (audited playlists):
{fingerprint as JSON}

Enriched track data:
{enriched_track_list as JSON}
```

Wait for the subagent to return. Extract the JSON block from its response.

---

## Step 6: Present the plan to the user

Show the analyst's full reasoning prose first.

Then summarize the proposed action:

- **reframe**: "Proposed: update description and mark as [ok]"
- **split**: List each cluster with name, track count, and description
- **purge**: List each track to remove with its reason
- **merge**: "Proposed: move all tracks into '{merge_into}' and mark this playlist as [merged]"

Then ask:
> Does this look right? You can approve, ask to adjust, or ask the analyst to reconsider.

Handle adjustments conversationally before proceeding.

---

## Step 7: Apply the confirmed action

### reframe
```
PUT /playlists/{id}  { "description": "<description>", "name": "[ok] {original_name}" }
```

### split
For each cluster:
```
POST /me/playlists  { "name": "{original_name} · {cluster_name}", "public": false, "description": "..." }
POST /playlists/{new_id}/items  { "uris": [...] }  (batches of 100)
```
Then rename the original:
```
PUT /playlists/{original_id}  { "name": "[split] {original_name}" }
```

### purge
```
DELETE /playlists/{id}/items  { "tracks": [{"uri": "..."}] }
PUT /playlists/{id}  { "description": "...", "name": "[ok] {original_name}" }
```

### merge
Look up the target playlist ID from `library-state.json` by matching the state's `name` field to `merge_into`. Use the stored `id` -- do not rely on the current Spotify display name, which may have been renamed during a prior audit.
```
POST /playlists/{target_id}/items  { "uris": [...all original tracks...] }  (batches of 100)
PUT /playlists/{original_id}  { "name": "[merged] {original_name}" }
```

---

## Step 8: Update library-state.json

Set the audited playlist entry:

```json
{
  "audit_status": "done",
  "action_taken": "reframe|split|purge|merge",
  "description_written": "<the description field from analyst output>",
  "sample_tracks": ["Track Name - Artist", ...]
}
```

For `sample_tracks`: format as "Track Name - Artist Name", up to 15 tracks. Source per action:
- **reframe**: any 15 tracks from the original track list
- **purge**: any 15 tracks from the tracks that remain after removal
- **split**: any 15 tracks from the largest cluster
- **merge**: leave `sample_tracks` as `[]` -- this playlist no longer exists as an independent entry; the target playlist retains its own samples

Update `last_updated` to the current ISO timestamp.

---

## Step 9: Update SESSION.md

Write a brief human-readable pickup note:

```markdown
# Session Log

## Last action
- Playlist: "{playlist_name}"
- Action: {reframe|split|purge|merge}
- Result: <one sentence>

## Next step
- Next pending playlist: "{next_pending_name}" (or "Audit complete" if none)
- Run `spotify:library-audit` to continue
```
