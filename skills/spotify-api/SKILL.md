---
name: spotify-api
description: >
  Spotify Web API skill for PKCE authentication and personal playlist
  management. Trigger when the user wants to authenticate with Spotify, list,
  create, or modify their own playlists, or add/remove tracks from them. Only
  works with playlists owned by the authenticated user.
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

Request only the scopes needed. For personal playlist read+write:

```
playlist-read-private playlist-modify-public playlist-modify-private
```

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
