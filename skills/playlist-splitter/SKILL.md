---
name: playlist-splitter
description: >
  Splits a Spotify playlist into a set of smaller, thematically coherent playlists using
  a music analyst subagent. Trigger when the user wants to break up, split, reorganize,
  or curate sub-playlists from an existing Spotify playlist.
---

# Playlist Splitter

Takes a Spotify playlist, deeply analyzes its tracks using a dedicated music analyst
subagent, and creates a set of thematically coherent sub-playlists in the user's account.

---

## Step 1: Conversational intake

Greet the user and ask for two things:

1. The Spotify playlist link or ID they want to split
2. Any preference on how to split (by mood, genre, decade, etc.) - frame this as optional:
   "or I can let the analyst decide based on what's in the playlist"

Extract the playlist ID from whatever the user provides:
- Full URL: `https://open.spotify.com/playlist/37i9dQZF1DXcBWIGoYBM5M` -> `37i9dQZF1DXcBWIGoYBM5M`
- URI: `spotify:playlist:37i9dQZF1DXcBWIGoYBM5M` -> `37i9dQZF1DXcBWIGoYBM5M`
- Raw ID: use as-is

Store any clustering preference the user expressed - it will be passed to the analyst.

---

## Step 2: Verify credentials

Check that the following environment variables are set before making any API calls:

- `SPOTIFY_CLIENT_ID`
- `SPOTIFY_CLIENT_SECRET`
- `SPOTIFY_ACCESS_TOKEN`

If any are missing, tell the user which ones are absent and stop. Refer them to the
`spotify-api` skill for setup instructions. Do not attempt to run the OAuth flow.

All API calls use:
```
Authorization: Bearer $SPOTIFY_ACCESS_TOKEN
```

Base URL: `https://api.spotify.com/v1`

Handle 429 responses by respecting the `Retry-After` header. On 401, inform the user
their access token has expired and they need to refresh it.

---

## Step 3: Fetch the source playlist

### 3a. Get playlist metadata

```
GET /playlists/{playlist_id}?fields=id,name,description,tracks.total
```

Store the playlist name - it will be used to prefix all created playlist names.

### 3b. Fetch all tracks (paginated)

```
GET /playlists/{playlist_id}/items?limit=50&offset=0
```

Use the `next` field to paginate until all tracks are collected. As of Feb 2026 the track
object is nested under the `item` key (not `track`). Accept either for compatibility:

```python
t = entry.get("item") or entry.get("track")
```

Extract from each item:
- `t.id`
- `t.uri`
- `t.name`
- `t.popularity`
- `t.explicit`
- `t.duration_ms`
- `t.artists[]` (id + name)
- `t.album` (id + name + release_date + album_type + total_tracks)

Skip any items where the resolved track object is null or has no `id` (local files,
removed tracks, or podcast episodes). Also check `t.type == "track"` to skip episodes.

---

## Step 4: Enrich with artist and album metadata

### 4a. Batch-fetch artist data

Collect all unique artist IDs from the tracks. Attempt batch fetch of up to 50 at a time:

```
GET /artists?ids={comma_separated_ids}
```

**Dev Mode caveat**: This endpoint returns 403 for Dev Mode apps. If the batch call fails
with 403, fall back to individual calls: `GET /artists/{id}`. Individual calls succeed but
return stripped data -- no `genres`, `popularity`, or `followers` fields. The analyst must
rely on its training knowledge for genre inference in this case.

For each artist store: `id`, `name`, `genres[]`, `popularity`, `followers.total`

### 4b. Batch-fetch album data

Collect all unique album IDs from the tracks. Attempt batch fetch of up to 20 at a time:

```
GET /albums?ids={comma_separated_ids}
```

**Dev Mode caveat**: Same as artists -- batch returns 403. Fall back to `GET /albums/{id}`
individually. Individual calls also return stripped data (no `genres`, `popularity`, or
`label` in Dev Mode).

For each album store: `id`, `name`, `release_date`, `album_type`, `genres[]`, `label`,
`total_tracks`, `popularity`

### 4c. Assemble enriched track list

Merge everything into a list of enriched track objects:

```json
{
  "uri": "spotify:track:6y0igZArWVi6Iz0rj35c1Y",
  "name": "Track Name",
  "popularity": 72,
  "explicit": false,
  "duration_ms": 217000,
  "artists": [
    {
      "id": "4Z8W4fKeB5YxbusRsdQVPb",
      "name": "Artist Name",
      "genres": ["indie rock", "alternative rock"],
      "popularity": 68,
      "followers": 2340000
    }
  ],
  "album": {
    "id": "6dVIqQ8qmQ5GBnJ9shOYss",
    "name": "Album Name",
    "release_date": "2019-03-01",
    "album_type": "album",
    "genres": [],
    "label": "XL Recordings",
    "total_tracks": 11,
    "popularity": 65
  }
}
```

---

## Step 5: Spawn the music analyst subagent

Read `ANALYST.md` from the same directory as this skill file. Use its full contents as
the system prompt for a subagent invoked via the Agent tool.

Pass the following as the subagent's task:

```
Source playlist: "{playlist_name}"
Total tracks: {N}
User clustering preference: {preference or "none - use your judgment"}

Enriched track data:
{enriched_track_list as JSON}
```

Wait for the subagent to return its full response. The output contains reasoning prose
followed by a JSON block delimited by ```json and ```.

Extract the JSON block from the subagent's response and parse it.

---

## Step 6: Present the plan to the user

Display the analyst's full reasoning prose so the user can read the rationale.

Then summarize the proposed clusters in a readable format:

```
Proposed split for "{playlist_name}":

1. {playlist_name} · {cluster_name} ({N} tracks)
   {cluster_description}

2. {playlist_name} · {cluster_name} ({N} tracks)
   {cluster_description}

...
```

Then ask:

> Does this look right? You can approve, ask me to rename any cluster, move specific
> tracks, merge clusters, or ask the analyst to rethink the split entirely.

Handle negotiation conversationally:
- **Rename**: update the cluster name in the plan
- **Move tracks**: update `track_uris` in the affected clusters
- **Merge**: combine two clusters, keeping the better name and merging descriptions
- **Re-run**: return to Step 5 with the user's feedback appended to the analyst prompt
- **Approve**: proceed to Step 7

Do not proceed to playlist creation until the user explicitly approves.

---

## Step 7: Create the playlists

### 7a. Create each playlist

For each approved cluster:

```
POST /me/playlists
{
  "name": "{playlist_name} · {cluster_name}",
  "public": false,
  "description": "{cluster_description}"
}
```

Note: `POST /users/{user_id}/playlists` returns 403 in Dev Mode apps. Use `POST /me/playlists` instead.

Store the returned playlist `id`.

### 7c. Add tracks in batches

For each created playlist, add its tracks in batches of 100:

```
POST /playlists/{playlist_id}/items
{
  "uris": ["{track_uri}", ...]
}
```

---

## Step 8: Report completion

Once all playlists are created, report:

```
Done! Created {N} playlists from "{playlist_name}":

- {playlist_name} · {cluster_name}: {M} tracks
- {playlist_name} · {cluster_name}: {M} tracks
...

All playlists are private. You can find them in your Spotify library.

Note: Spotify's API doesn't support creating folders. Your playlists share the
"{playlist_name} ·" prefix so they sort together in your library. You can manually
drag them into a folder in the Spotify desktop app if you'd like.
```
