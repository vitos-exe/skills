---
name: spotify-api
description: >
  Spotify Web API skill for PKCE authentication, playlist management, track
  enrichment, and Liked Songs access. Trigger when the user wants to
  authenticate with Spotify, list/create/modify playlists, add/remove tracks,
  fetch artist or album metadata, or read their Liked Songs library.
---

# Spotify Web API: Auth + Playlists

Handles PKCE authentication and playlist management.

**Critical rule**: all scripts run inline via the Bash tool using a heredoc
(`python3 << 'EOF' ... EOF`). Never use the Write tool to create `.py` files.

Base URL: `https://api.spotify.com/v1`

---

## Step 1: Credentials check

Required env vars:

- `SPOTIFY_CLIENT_ID` — if missing, stop and ask the user to set it
- `SPOTIFY_ACCESS_TOKEN` — if missing, run the PKCE flow below

`SPOTIFY_CLIENT_SECRET` is not required for PKCE.

---

## Step 2: PKCE auth (run inline when token is missing)

Request only the scopes needed for the task at hand. Common scope sets:

| Use case | Scopes |
|----------|--------|
| Playlist read + write | `playlist-read-private playlist-read-collaborative playlist-modify-public playlist-modify-private` |
| Liked Songs | `user-library-read` |
| Full library management | `playlist-read-private playlist-read-collaborative playlist-modify-public playlist-modify-private user-library-read` |

Run this inline. Use `127.0.0.1` not `localhost` as the redirect host.

```bash
python3 << 'EOF'
import hashlib, base64, os, socket, urllib.parse, webbrowser, requests

CLIENT_ID   = os.environ["SPOTIFY_CLIENT_ID"]
REDIRECT    = "http://127.0.0.1:8888/callback"
SCOPES      = "playlist-read-private playlist-modify-public playlist-modify-private"

verifier  = base64.urlsafe_b64encode(os.urandom(32)).rstrip(b"=").decode()
challenge = base64.urlsafe_b64encode(hashlib.sha256(verifier.encode()).digest()).rstrip(b"=").decode()
state     = base64.urlsafe_b64encode(os.urandom(16)).rstrip(b"=").decode()

auth_url = "https://accounts.spotify.com/authorize?" + urllib.parse.urlencode({
    "client_id": CLIENT_ID, "response_type": "code",
    "redirect_uri": REDIRECT, "scope": SCOPES,
    "state": state, "code_challenge_method": "S256", "code_challenge": challenge,
})

sock = socket.create_server(("127.0.0.1", 8888))
webbrowser.open(auth_url)
conn, _ = sock.accept()
raw = conn.recv(4096).decode()
conn.sendall(b"HTTP/1.1 200 OK\r\n\r\nAuth complete. Return to terminal.")
conn.close()
code = urllib.parse.parse_qs(urllib.parse.urlparse(raw.split()[1]).query)["code"][0]

r = requests.post("https://accounts.spotify.com/api/token", data={
    "grant_type": "authorization_code", "code": code,
    "redirect_uri": REDIRECT, "client_id": CLIENT_ID, "code_verifier": verifier,
})
r.raise_for_status()
data = r.json()
print(f"SPOTIFY_ACCESS_TOKEN={data['access_token']}")
print(f"SPOTIFY_REFRESH_TOKEN={data['refresh_token']}")
EOF
```

Capture the printed tokens. Tell the user to save the refresh token for future sessions, then continue.

---

## Step 3: Token refresh (when you get a 401)

```bash
python3 << 'EOF'
import os, requests
r = requests.post("https://accounts.spotify.com/api/token", data={
    "grant_type": "refresh_token",
    "refresh_token": os.environ["SPOTIFY_REFRESH_TOKEN"],
    "client_id": os.environ["SPOTIFY_CLIENT_ID"],
})
r.raise_for_status()
data = r.json()
print(f"SPOTIFY_ACCESS_TOKEN={data['access_token']}")
if "refresh_token" in data:
    print(f"SPOTIFY_REFRESH_TOKEN={data['refresh_token']}")
EOF
```

Update `SPOTIFY_ACCESS_TOKEN` in the environment, then retry the failed call.

---

## Step 4: Making requests (inline pattern)

Use this pattern for every API call. Respect 429 with `Retry-After`.

```python
import os, time, requests

def spotify(method, path, *, params=None, json=None):
    token = os.environ["SPOTIFY_ACCESS_TOKEN"]
    for attempt in range(3):
        r = requests.request(method, f"https://api.spotify.com/v1{path}",
                             headers={"Authorization": f"Bearer {token}"},
                             params=params, json=json)
        if r.status_code == 204:
            return None
        if r.status_code == 429:
            time.sleep(int(r.headers.get("Retry-After", 1)) * 2 ** attempt)
            continue
        r.raise_for_status()
        return r.json() if r.content else None
    raise RuntimeError(f"Request failed after 3 attempts: {path}")
```

Embed this helper at the top of every inline script that needs it.

---

## Step 5: Playlist operations

### List the user's playlists (paginated)

Note: playlist objects returned by this endpoint do **not** include a `tracks` key.
To get track count or items, fetch the individual playlist via `GET /playlists/{id}`.

```python
def get_my_playlists():
    playlists, path, params = [], "/me/playlists", {"limit": 50, "offset": 0}
    while path:
        page = spotify("GET", path, params=params)
        playlists.extend(page["items"])
        next_url = page.get("next")
        path   = next_url[len("https://api.spotify.com/v1"):] if next_url else None
        params = {}
    return playlists
```

### Get playlist metadata

```python
playlist = spotify("GET", f"/playlists/{playlist_id}")
name = playlist["name"]
total = playlist.get("tracks", {}).get("total")
```

### Fetch all playlist items (paginated)

As of Feb 2026 use `/items`, not `/tracks`. The track object is under `item` key (fall back to `track` for older data).

```python
def get_items(playlist_id):
    tracks, path, params = [], f"/playlists/{playlist_id}/items", {"limit": 50, "offset": 0}
    while path:
        page = spotify("GET", path, params=params)
        for entry in page["items"]:
            t = entry.get("item") or entry.get("track")
            if t and t.get("id") and t.get("type") == "track":
                tracks.append(t)
        next_url = page.get("next")
        path   = next_url[len("https://api.spotify.com/v1"):] if next_url else None
        params = {}
    return tracks
```

### Create a playlist

```python
# POST /users/{id}/playlists returns 403 in Dev Mode — use /me/playlists
playlist = spotify("POST", "/me/playlists", json={
    "name": "My Playlist",
    "public": False,
    "description": "Created via API",
})
playlist_id = playlist["id"]
```

### Add tracks (max 100 per call)

```python
uris = ["spotify:track:6y0igZArWVi6Iz0rj35c1Y", ...]
for i in range(0, len(uris), 100):
    spotify("POST", f"/playlists/{playlist_id}/items", json={"uris": uris[i:i+100]})
```

### Remove tracks

```python
spotify("DELETE", f"/playlists/{playlist_id}/items", json={
    "tracks": [{"uri": u} for u in uris]
})
```

### Update playlist details

```python
spotify("PUT", f"/playlists/{playlist_id}", json={
    "name": "New Name",          # optional
    "description": "New desc",   # optional
    "public": False,             # optional
})
```

---

## Step 6: Liked Songs and track enrichment

### Fetch Liked Songs (paginated)

Requires scope: `user-library-read`.

```python
def get_liked_songs():
    tracks, path, params = [], "/me/tracks", {"limit": 50, "offset": 0}
    while path:
        page = spotify("GET", path, params=params)
        for entry in page["items"]:
            t = entry.get("track")
            if t and t.get("id") and t.get("type") == "track":
                tracks.append(t)
        next_url = page.get("next")
        path   = next_url[len("https://api.spotify.com/v1"):] if next_url else None
        params = {}
    return tracks
```

Use these to enrich a track list with artist genres/popularity and album metadata before passing to an analyst subagent.

### Batch-fetch artist data (up to 50 per call)

```python
def get_artists(artist_ids):
    artists = {}
    ids = list(dict.fromkeys(artist_ids))  # deduplicate, preserve order
    for i in range(0, len(ids), 50):
        batch = ids[i:i+50]
        try:
            data = spotify("GET", "/artists", params={"ids": ",".join(batch)})
            for a in data["artists"]:
                if a:
                    artists[a["id"]] = a
        except requests.HTTPError as e:
            if e.response.status_code == 403:
                # Dev Mode: batch endpoint blocked, fall back to individual calls
                for aid in batch:
                    try:
                        a = spotify("GET", f"/artists/{aid}")
                        artists[a["id"]] = a
                    except Exception:
                        pass
            else:
                raise
    return artists
```

Dev Mode note: batch returns 403. Individual calls succeed but return stripped data (no `genres`, `popularity`, or `followers`). Analyst must rely on training knowledge for genre inference.

### Batch-fetch album data (up to 20 per call)

```python
def get_albums(album_ids):
    albums = {}
    ids = list(dict.fromkeys(album_ids))
    for i in range(0, len(ids), 20):
        batch = ids[i:i+20]
        try:
            data = spotify("GET", "/albums", params={"ids": ",".join(batch)})
            for a in data["albums"]:
                if a:
                    albums[a["id"]] = a
        except requests.HTTPError as e:
            if e.response.status_code == 403:
                for aid in batch:
                    try:
                        a = spotify("GET", f"/albums/{aid}")
                        albums[a["id"]] = a
                    except Exception:
                        pass
            else:
                raise
    return albums
```

Dev Mode note: same 403 issue as artists. Individual calls also return stripped data (no `genres`, `popularity`, or `label`).

### Assemble enriched track list

After fetching artists and albums, merge into enriched objects:

```python
def enrich_tracks(tracks, artists_map, albums_map):
    enriched = []
    for t in tracks:
        enriched.append({
            "uri": t["uri"],
            "name": t["name"],
            "popularity": t.get("popularity"),
            "explicit": t.get("explicit"),
            "duration_ms": t.get("duration_ms"),
            "artists": [
                {
                    "id": a["id"],
                    "name": a["name"],
                    "genres": artists_map.get(a["id"], {}).get("genres", []),
                    "popularity": artists_map.get(a["id"], {}).get("popularity"),
                    "followers": artists_map.get(a["id"], {}).get("followers", {}).get("total"),
                }
                for a in t.get("artists", [])
            ],
            "album": {
                "id": t["album"]["id"],
                "name": t["album"]["name"],
                "release_date": t["album"].get("release_date"),
                "album_type": t["album"].get("album_type"),
                "genres": albums_map.get(t["album"]["id"], {}).get("genres", []),
                "label": albums_map.get(t["album"]["id"], {}).get("label"),
                "total_tracks": t["album"].get("total_tracks"),
                "popularity": albums_map.get(t["album"]["id"], {}).get("popularity"),
            },
        })
    return enriched
```

---

## February 2026 changes (playlists)

| Old | New |
|-----|-----|
| `GET/POST/PUT/DELETE /playlists/{id}/tracks` | `/playlists/{id}/items` |
| `items[].track` key | `items[].item` key (accept both) |
| `POST /users/{id}/playlists` | `POST /me/playlists` |
| Dev Mode: 25 users | Dev Mode: 5 users |
| Dev Mode: any owner account | Dev Mode: owner needs active Premium |

---

## Error reference

| Status | Cause | Action |
|--------|-------|--------|
| 401 | Token expired | Run Step 3 refresh, retry |
| 403 "Insufficient client scope" | Missing scope | Re-authorize with correct scopes |
| 403 on `POST /users/{id}/playlists` | Dev Mode restriction | Use `POST /me/playlists` |
| 404 | Resource not found | Check playlist/track ID |
| 429 | Rate limited | Sleep `Retry-After` seconds |
