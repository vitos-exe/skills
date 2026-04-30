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
  "reroute_playlist_id": null,
  "playlists": [
    {
      "id": "...",
      "original_id": null,
      "name": "...",
      "track_count": null,
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

Tell the user how many playlists were found and that the audit is starting. Then do a second pass: call `GET /playlists/{id}` for each playlist and store the track count in `track_count`. This lets the summary report size distribution (e.g. "54 playlists: 12 with 50+ tracks, 8 with fewer than 10") and helps the user set `mess_score` priorities.

**If exists with pending entries**: first run the migration check below, then pick the next target.

**Migration check**: Scan all `done` playlists. If any have `original_id == null` AND a `name`
that does not start with `[agent]`, they were audited before the agent-copy convention was
introduced (modified in place). For each:
1. `PUT /playlists/{id} { "name": "[agent] {current_name}" }` -- rename on Spotify
2. Update the `name` field in `library-state.json` to `[agent] {current_name}`
3. Set `original_id` to `null` -- the original was modified in place; no separate original exists

Then pick the next target by `mess_score` descending (null scores = pick first in list order). The user can override by naming a specific playlist. If the user asks to skip the current playlist, set its `audit_status` to `"skipped"` and pick the next pending one.

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

- **reframe**: "Proposed: update description and mark as [agent] [ok]"
- **split**: List each cluster with name, track count, and description
- **purge**: List each track to remove with its reason
- **merge**: "Proposed: move all tracks into '{merge_into}' and mark this playlist as [agent] [merged]"

Then ask:
> Does this look right? You can approve, ask to adjust, or ask the analyst to reconsider.

Handle adjustments conversationally before proceeding.

---

## Step 7: Apply the confirmed action

**Description format**: `description_written` uses `\n` between the four lines. When passing to the Spotify API, replace `\n` with ` | ` -- Spotify rejects newlines in descriptions (returns 400). Keep the `\n` version in `library-state.json`.

### Create agent copy (all actions)

Before applying any action, create an agent-managed copy of the original playlist. The original
is never modified.

1. Read `id` from the state entry and treat it as `original_id` for this step.
2. `POST /me/playlists { "name": "[agent] {base_name}", "public": false, "description": "" }`
   where `base_name` is the `name` field from the state entry -- used directly, no stripping
   needed since pending playlists never carry audit tags.
   Save the returned playlist `id` as `copy_id`.
3. `GET /playlists/{original_id}/items` (paginated) -- collect all track URIs.
4. `POST /playlists/{copy_id}/items { "uris": [...] }` (batches of 100) -- populate the copy.

All subsequent writes in this step target `copy_id`. The original playlist at `original_id`
is left untouched.

### reframe
```
PUT /playlists/{copy_id}  { "description": "<description>", "name": "[agent] [ok] {base_name}" }
```

### split
Rename the copy first:
```
PUT /playlists/{copy_id}  { "name": "[agent] [split] {base_name}" }
```
Then create each cluster as a new agent playlist:
```
POST /me/playlists  { "name": "[agent] {base_name} · {cluster_name}", "public": false, "description": "..." }
POST /playlists/{new_id}/items  { "uris": [...] }  (batches of 100)
```

### purge
Before removing tracks, preserve them in the `[agent] [reroute]` holding playlist (`PUT /me/tracks` is blocked in Dev Mode):

1. If `reroute_playlist_id` is null in state: `POST /me/playlists { "name": "[agent] [reroute]", "public": false }` and save the returned ID to `reroute_playlist_id` in state.
2. `POST /playlists/{reroute_id}/items  { "uris": [...purged URIs...] }  (batches of 100)`
3. `DELETE /playlists/{copy_id}/items  { "items": [{"uri": "..."}] }`
4. `PUT /playlists/{copy_id}  { "description": "...", "name": "[agent] [ok] {base_name}" }`

`[agent] [reroute]` is a permanent holding area -- Phase 2 (`spotify:sort-inbox`) will route these tracks to the right playlists.

### merge
Look up the target playlist's `id` from `library-state.json` by matching the state's `name` field to `merge_into`. Use the stored `id` (the agent copy ID) -- do not rely on the current Spotify display name.
```
POST /playlists/{target_id}/items  { "uris": [...all copy tracks...] }  (batches of 100)
PUT /playlists/{copy_id}  { "name": "[agent] [merged] {base_name}" }
```

---

## Step 8: Update library-state.json

Set the audited playlist entry:

```json
{
  "id": "<copy_id -- the agent copy's playlist ID>",
  "original_id": "<original_id -- the untouched original's playlist ID>",
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

Do not update the `name` field -- it stays as the original plain name (e.g. `house`). This is
intentional: the fingerprint and merge lookup use `name` for human-readable matching; the
Spotify display name diverges and that is fine.

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

---

## Batch mode (optional)

When the user wants to process multiple playlists in one session, the fetch-enrich-analyze steps can be parallelized:

1. Pick N playlists to process (e.g. all pending, or a named subset).
2. For each: fetch + enrich tracks (Steps 3-4), then spawn an analyst subagent with `run_in_background=true`.
3. Wait for all subagents to complete, then present proposals sequentially for user approval.
4. Apply each confirmed action one at a time (Steps 7-9).

**Trade-off**: all parallel analysts share the same library fingerprint snapshot at the time they were spawned. None can see each other as merge candidates. This is fine when the batch playlists are clearly distinct; avoid it when you suspect duplicates in the batch.
