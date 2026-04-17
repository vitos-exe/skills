---
name: spotify-api
description: >
  Comprehensive Spotify Web API skill. Trigger when the user wants to integrate with
  Spotify: authenticate users, search tracks/albums/artists, manage playlists, control
  playback, read user library, fetch top items, or build any Spotify-powered feature.
  Covers Authorization Code (PKCE), Client Credentials, scopes, rate limiting, pagination,
  February 2026 API changes, and practical code patterns in Python and TypeScript.
---

# Spotify Web API

This skill guides you through building correct, production-ready Spotify integrations.
It covers authentication, the full API surface, rate limiting, pagination, and the
significant February 2026 API changes that affect every app.

---

## Step 1: Clarify scope and choose an auth flow

Before writing any code, determine:

1. **Does the feature need user data or playback control?**
   - Yes (playlists, library, top items, player) → Authorization Code Flow (or PKCE)
   - No (catalog search, track/album/artist metadata only) → Client Credentials Flow

2. **Where does the code run?**
   - Server with a secret → Authorization Code Flow (standard)
   - Browser SPA, mobile app, CLI, Electron → Authorization Code + PKCE (no secret)
   - Backend service only → Client Credentials

3. **What quota mode is the Spotify app registered under?**
   - **Development Mode** (default): max 5 users, reduced search limits (max 10 results), no browse/catalog endpoints
   - **Extended Quota Mode**: full user limit, all endpoints available

Ask the user which language they want (Python or TypeScript are the primary examples here).

---

## Step 2: Set up a Spotify Developer App

If the user does not yet have a Spotify app:

1. Go to https://developer.spotify.com/dashboard and create an app
2. Note the **Client ID** and **Client Secret**
3. Add a Redirect URI in app settings (e.g., `http://localhost:8888/callback`)
4. Store credentials in environment variables:
   ```
   SPOTIFY_CLIENT_ID=...
   SPOTIFY_CLIENT_SECRET=...
   SPOTIFY_REDIRECT_URI=http://localhost:8888/callback
   ```

Important: the app owner must have an active Spotify Premium account for Development Mode to function (required since February 2026).

---

## Step 3: Implement authentication

### Authorization Code Flow + PKCE (browser/mobile/CLI)

PKCE eliminates the need to store a client secret on untrusted clients.

**Python implementation:**

```python
import hashlib
import base64
import os
import urllib.parse
import requests

CLIENT_ID = os.environ["SPOTIFY_CLIENT_ID"]
REDIRECT_URI = os.environ["SPOTIFY_REDIRECT_URI"]
TOKEN_URL = "https://accounts.spotify.com/api/token"
AUTH_URL = "https://accounts.spotify.com/authorize"

def generate_pkce():
    verifier = base64.urlsafe_b64encode(os.urandom(32)).rstrip(b"=").decode()
    digest = hashlib.sha256(verifier.encode()).digest()
    challenge = base64.urlsafe_b64encode(digest).rstrip(b"=").decode()
    return verifier, challenge

def build_auth_url(scopes: list[str]) -> tuple[str, str]:
    verifier, challenge = generate_pkce()
    state = base64.urlsafe_b64encode(os.urandom(16)).rstrip(b"=").decode()
    params = {
        "client_id": CLIENT_ID,
        "response_type": "code",
        "redirect_uri": REDIRECT_URI,
        "scope": " ".join(scopes),
        "state": state,
        "code_challenge_method": "S256",
        "code_challenge": challenge,
    }
    url = AUTH_URL + "?" + urllib.parse.urlencode(params)
    return url, verifier  # store verifier server-side, keyed by state

def exchange_code(code: str, verifier: str) -> dict:
    r = requests.post(TOKEN_URL, data={
        "grant_type": "authorization_code",
        "code": code,
        "redirect_uri": REDIRECT_URI,
        "client_id": CLIENT_ID,
        "code_verifier": verifier,
    })
    r.raise_for_status()
    return r.json()  # { access_token, refresh_token, expires_in }
```

### Authorization Code Flow (server-side with secret)

```python
import base64

def exchange_code_server(code: str) -> dict:
    credentials = base64.b64encode(
        f"{os.environ['SPOTIFY_CLIENT_ID']}:{os.environ['SPOTIFY_CLIENT_SECRET']}".encode()
    ).decode()
    r = requests.post(TOKEN_URL, headers={"Authorization": f"Basic {credentials}"}, data={
        "grant_type": "authorization_code",
        "code": code,
        "redirect_uri": REDIRECT_URI,
    })
    r.raise_for_status()
    return r.json()
```

### Client Credentials Flow (no user context)

```python
def get_app_token() -> str:
    credentials = base64.b64encode(
        f"{os.environ['SPOTIFY_CLIENT_ID']}:{os.environ['SPOTIFY_CLIENT_SECRET']}".encode()
    ).decode()
    r = requests.post(TOKEN_URL, headers={"Authorization": f"Basic {credentials}"}, data={
        "grant_type": "client_credentials",
    })
    r.raise_for_status()
    return r.json()["access_token"]
```

### Token refresh

```python
import time

class SpotifyTokenManager:
    def __init__(self, access_token: str, refresh_token: str, expires_in: int):
        self.access_token = access_token
        self.refresh_token = refresh_token
        self._expires_at = time.time() + expires_in - 120  # refresh 2 min early

    def get_token(self) -> str:
        if time.time() >= self._expires_at:
            self._refresh()
        return self.access_token

    def _refresh(self):
        r = requests.post(TOKEN_URL, data={
            "grant_type": "refresh_token",
            "refresh_token": self.refresh_token,
            "client_id": CLIENT_ID,
        })
        r.raise_for_status()
        data = r.json()
        self.access_token = data["access_token"]
        self._expires_at = time.time() + data["expires_in"] - 120
        if "refresh_token" in data:  # Spotify may rotate it
            self.refresh_token = data["refresh_token"]
```

---

## Step 4: Build a base API client with rate limit handling

All Spotify API calls need consistent error handling and rate limit respect.

```python
import time
import requests

BASE_URL = "https://api.spotify.com/v1"

class SpotifyClient:
    def __init__(self, token_manager):
        self._tokens = token_manager

    def _headers(self) -> dict:
        return {"Authorization": f"Bearer {self._tokens.get_token()}"}

    def get(self, path: str, **params) -> dict:
        return self._request("GET", path, params=params)

    def post(self, path: str, json=None) -> dict | None:
        return self._request("POST", path, json=json)

    def put(self, path: str, json=None) -> None:
        self._request("PUT", path, json=json)

    def delete(self, path: str, json=None) -> None:
        self._request("DELETE", path, json=json)

    def _request(self, method: str, path: str, retries: int = 3, **kwargs):
        url = BASE_URL + path
        for attempt in range(retries):
            r = requests.request(method, url, headers=self._headers(), **kwargs)
            if r.status_code == 204:
                return None
            if r.status_code == 429:
                wait = int(r.headers.get("Retry-After", 1))
                time.sleep(wait * (2 ** attempt))  # respect + backoff
                continue
            if r.status_code == 401 and attempt == 0:
                self._tokens._refresh()  # force refresh once
                continue
            r.raise_for_status()
            if r.content:
                return r.json()
        raise RuntimeError(f"Request to {path} failed after {retries} attempts")
```

**TypeScript equivalent (using fetch):**

```typescript
const BASE_URL = "https://api.spotify.com/v1";

async function spotifyRequest(
  method: string,
  path: string,
  token: string,
  body?: unknown,
  attempt = 0
): Promise<unknown> {
  const res = await fetch(`${BASE_URL}${path}`, {
    method,
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
    },
    body: body ? JSON.stringify(body) : undefined,
  });

  if (res.status === 204) return null;
  if (res.status === 429 && attempt < 3) {
    const retryAfter = Number(res.headers.get("Retry-After") ?? 1);
    await new Promise((r) => setTimeout(r, retryAfter * 1000 * 2 ** attempt));
    return spotifyRequest(method, path, token, body, attempt + 1);
  }
  if (!res.ok) throw new Error(`Spotify ${res.status}: ${await res.text()}`);
  return res.json();
}
```

---

## Step 5: Select required scopes

Request only the scopes you actually need. Present this table to the user when relevant:

| Scope | What it enables |
|---|---|
| `user-read-private` | Current user profile (id, display name, country, subscription) |
| `user-read-email` | User email address |
| `user-library-read` | Read saved tracks, albums, shows |
| `user-library-modify` | Save/remove items from library |
| `playlist-read-private` | Read private playlists |
| `playlist-read-collaborative` | Read collaborative playlists |
| `playlist-modify-public` | Create/edit public playlists |
| `playlist-modify-private` | Create/edit private playlists |
| `user-read-playback-state` | Read current playback (device, track, position) |
| `user-modify-playback-state` | Play, pause, skip, seek, volume, shuffle, repeat |
| `user-read-currently-playing` | Currently playing track only |
| `user-read-recently-played` | Recently played history |
| `user-top-read` | User's top tracks and artists |
| `user-follow-read` | Followed artists/users |
| `user-follow-modify` | Follow/unfollow artists/users |

A 403 Forbidden error means a missing scope - check the required scope in the API reference for that endpoint.

---

## Step 6: Common API operations

### Search

```python
def search(client: SpotifyClient, query: str, types: list[str], market: str = "from_token", limit: int = 10):
    # types: "track", "album", "artist", "playlist", "show", "episode", "audiobook"
    # limit max is 10 in Dev Mode (was 50 before Feb 2026)
    return client.get("/search", q=query, type=",".join(types), market=market, limit=limit)

# Field filters in query string:
# artist:radiohead year:2000-2010
# track:karma artist:taylor swift
# genre:jazz tag:hipster  (obscure content)
# tag:new  (recently released)
```

### Get current user profile

```python
def get_me(client: SpotifyClient) -> dict:
    return client.get("/me")  # requires user-read-private
```

### User's top tracks/artists

```python
def get_top(client: SpotifyClient, item_type: str, time_range: str = "medium_term", limit: int = 20) -> dict:
    # item_type: "tracks" or "artists"
    # time_range: "short_term" (4 weeks), "medium_term" (6 months), "long_term" (all time)
    return client.get(f"/me/top/{item_type}", time_range=time_range, limit=limit)
```

### Playlists (updated endpoints - use /items not /tracks)

```python
def get_playlist(client: SpotifyClient, playlist_id: str) -> dict:
    return client.get(f"/playlists/{playlist_id}")

def get_playlist_items(client: SpotifyClient, playlist_id: str) -> list[dict]:
    # /items replaces /tracks as of Feb 2026
    # Each element now has an "item" key (not "track") — accept both for safety
    raw = []
    url = f"/playlists/{playlist_id}/items"
    params = {"limit": 50, "offset": 0}
    while url:
        page = client.get(url if url.startswith("/") else url.replace(BASE_URL, ""), **params)
        raw.extend(page["items"])
        url = page.get("next")
        params = {}  # next URL includes all params
    tracks = []
    for entry in raw:
        t = entry.get("item") or entry.get("track")
        if t and t.get("id") and t.get("type") == "track":
            tracks.append(t)
    return tracks

def create_playlist(client: SpotifyClient, name: str, public: bool = False, description: str = "") -> dict:
    # POST /users/{id}/playlists returns 403 in Dev Mode apps — use /me/playlists instead
    return client.post("/me/playlists", json={
        "name": name,
        "public": public,
        "description": description,
    })

def add_to_playlist(client: SpotifyClient, playlist_id: str, uris: list[str], position: int | None = None) -> dict:
    # uris: list of Spotify track URIs, e.g. ["spotify:track:6y0igZArWVi6Iz0rj35c1Y"]
    # max 100 uris per request
    body = {"uris": uris}
    if position is not None:
        body["position"] = position
    return client.post(f"/playlists/{playlist_id}/items", json=body)

def remove_from_playlist(client: SpotifyClient, playlist_id: str, uris: list[str]):
    client.delete(f"/playlists/{playlist_id}/items", json={
        "tracks": [{"uri": u} for u in uris]
    })

def update_playlist_details(client: SpotifyClient, playlist_id: str, **kwargs):
    # kwargs: name, description, public, collaborative
    client.put(f"/playlists/{playlist_id}", json=kwargs)
```

### Tracks, Albums, Artists (batch fetching)

```python
def get_tracks(client: SpotifyClient, track_ids: list[str]) -> list[dict]:
    # max 50 ids per call
    results = []
    for i in range(0, len(track_ids), 50):
        batch = ",".join(track_ids[i:i+50])
        data = client.get("/tracks", ids=batch)
        results.extend(data["tracks"])
    return results

def get_albums(client: SpotifyClient, album_ids: list[str]) -> list[dict]:
    results = []
    for i in range(0, len(album_ids), 20):  # max 20 for albums
        batch = ",".join(album_ids[i:i+20])
        data = client.get("/albums", ids=batch)
        results.extend(data["albums"])
    return results

def get_artist_albums(client: SpotifyClient, artist_id: str, include_groups: str = "album,single") -> list[dict]:
    albums = []
    params = {"include_groups": include_groups, "limit": 50, "offset": 0}
    while True:
        page = client.get(f"/artists/{artist_id}/albums", **params)
        albums.extend(page["items"])
        if page["next"] is None:
            break
        params["offset"] += params["limit"]
    return albums
```

### User library

```python
def save_items(client: SpotifyClient, uris: list[str]):
    # Generic library endpoint (Feb 2026+) - accepts any Spotify URI type
    # requires user-library-modify
    client.put("/me/library", json={"uris": uris})

def remove_items(client: SpotifyClient, uris: list[str]):
    client.delete("/me/library", json={"uris": uris})

def check_saved(client: SpotifyClient, uris: list[str]) -> list[bool]:
    return client.get("/me/library/contains", uris=",".join(uris))

def get_saved_tracks(client: SpotifyClient) -> list[dict]:
    tracks = []
    params = {"limit": 50, "offset": 0}
    while True:
        page = client.get("/me/tracks", **params)
        tracks.extend(page["items"])
        if page["next"] is None:
            break
        params["offset"] += params["limit"]
    return tracks
```

### Playback control (requires Premium)

```python
def get_playback_state(client: SpotifyClient) -> dict | None:
    return client.get("/me/player")  # returns None if no active device

def get_devices(client: SpotifyClient) -> list[dict]:
    return client.get("/me/player/devices")["devices"]

def play(client: SpotifyClient, device_id: str | None = None,
         context_uri: str | None = None, uris: list[str] | None = None,
         offset: dict | None = None, position_ms: int = 0):
    # context_uri: "spotify:playlist:...", "spotify:album:...", "spotify:artist:..."
    # uris: list of track URIs (alternative to context_uri)
    # offset: {"position": 0} or {"uri": "spotify:track:..."}
    params = {}
    if device_id:
        params["device_id"] = device_id
    body = {}
    if context_uri:
        body["context_uri"] = context_uri
    if uris:
        body["uris"] = uris
    if offset:
        body["offset"] = offset
    body["position_ms"] = position_ms
    client.put("/me/player/play", json=body)

def pause(client: SpotifyClient, device_id: str | None = None):
    params = f"?device_id={device_id}" if device_id else ""
    client.put(f"/me/player/pause{params}")

def skip_next(client: SpotifyClient):
    client.post("/me/player/next")

def skip_previous(client: SpotifyClient):
    client.post("/me/player/previous")

def seek(client: SpotifyClient, position_ms: int, device_id: str | None = None):
    params = {"position_ms": position_ms}
    if device_id:
        params["device_id"] = device_id
    client.put("/me/player/seek", **params)

def set_volume(client: SpotifyClient, volume_percent: int):
    # 0-100
    client.put("/me/player/volume", volume_percent=volume_percent)

def set_shuffle(client: SpotifyClient, state: bool):
    client.put("/me/player/shuffle", state=str(state).lower())

def set_repeat(client: SpotifyClient, state: str):
    # state: "off", "context" (repeat playlist/album), "track" (repeat one)
    client.put("/me/player/repeat", state=state)

def get_currently_playing(client: SpotifyClient) -> dict | None:
    return client.get("/me/player/currently-playing")
```

---

## Step 7: Pagination patterns

### Offset-based (most endpoints)

```python
def paginate_offset(client: SpotifyClient, path: str, limit: int = 50, **params) -> list:
    results = []
    offset = 0
    while True:
        page = client.get(path, limit=limit, offset=offset, **params)
        results.extend(page["items"])
        if page["next"] is None:
            break
        offset += limit
    return results
```

### Cursor-based (recently played, some user endpoints)

```python
def get_recently_played(client: SpotifyClient, limit: int = 50, after: int | None = None) -> dict:
    # after: Unix timestamp in ms - returns items played after this time
    # Returns: { items, cursors: { after, before }, next }
    params = {"limit": limit}
    if after:
        params["after"] = after
    return client.get("/me/player/recently-played", **params)

def paginate_cursor(client: SpotifyClient, path: str, limit: int = 50) -> list:
    results = []
    cursor = None
    while True:
        params = {"limit": limit}
        if cursor:
            params["after"] = cursor
        page = client.get(path, **params)
        results.extend(page["items"])
        if not page.get("next"):
            break
        cursor = page["cursors"]["after"]
    return results
```

---

## Step 8: Spotify URIs and IDs

**URI format:** `spotify:{type}:{id}`
- `spotify:track:6y0igZArWVi6Iz0rj35c1Y`
- `spotify:playlist:37i9dQZF1DXcBWIGoYBM5M`
- `spotify:artist:4Z8W4fKeB5YxbusRsdQVPb`
- `spotify:album:6dVIqQ8qmQ5GBnJ9shOYss`

**ID format:** 22-character base62 string. Extract from URI: `uri.split(":")[2]`

The new generic `/me/library` endpoint (Feb 2026) accepts URIs instead of separate IDs for each type.

---

## Step 9: February 2026 changes (must know)

These changes affect all apps - check before implementing deprecated patterns:

| Area | Old | New / Status |
|---|---|---|
| Playlist items endpoint | `GET/POST/PUT/DELETE /playlists/{id}/tracks` | Use `/playlists/{id}/items` |
| Playlist items key | `items[].track` | `items[].item` (fall back to `track` for safety) |
| Create playlist | `POST /users/{id}/playlists` | Use `POST /me/playlists` (user endpoint returns 403 in Dev Mode) |
| User library | Type-specific save/remove/check endpoints | Generic `PUT/DELETE/GET /me/library` with URIs |
| Search limit | max 50, default 20 | max 10, default 5 (Dev Mode) |
| Batch episodes | `GET /episodes?ids=...` | Removed |
| Batch shows | `GET /shows?ids=...` | Removed |
| Artist top tracks | `GET /artists/{id}/top-tracks` | Removed |
| Public user profile | `GET /users/{id}` | Removed |
| Recommendations | `GET /recommendations` | Deprecated (Nov 2024), unavailable for new apps |
| Audio features | `GET /audio-features` | Deprecated (Nov 2024), unavailable for new apps |
| Dev Mode users | 25 users | 5 users |
| Dev Mode owner | Any account | Must have active Premium |
| Dev Mode catalog batch | `GET /artists?ids=`, `GET /albums?ids=` work | Return 403 — use individual `GET /artists/{id}` / `GET /albums/{id}` calls instead |
| Dev Mode catalog fields | Full artist/album objects | Individual endpoints return stripped objects: no `genres`, `popularity`, or `followers` |

If you encounter any code using the old endpoints, migrate immediately - old endpoints return 404 or 403 for new apps.

---

## Step 10: Error handling reference

```python
from requests import HTTPError

def handle_spotify_error(e: HTTPError):
    status = e.response.status_code
    if status == 400:
        # Malformed request - fix the parameters
        raise ValueError(f"Bad request: {e.response.json().get('error', {}).get('message')}")
    elif status == 401:
        # Token expired or invalid - refresh and retry (handled in client)
        pass
    elif status == 403:
        # Wrong scope or Premium required - do not retry
        msg = e.response.json().get("error", {}).get("message", "")
        raise PermissionError(f"Forbidden: {msg}. Check required scopes or Premium status.")
    elif status == 404:
        # Resource doesn't exist
        raise LookupError("Resource not found")
    elif status == 429:
        # Rate limited - handled in client with Retry-After
        pass
    elif status >= 500:
        # Spotify server error - retry with backoff
        pass
```

**Common error causes:**
- 401: access token expired (refresh) or client credentials flow accessing user endpoint (switch flow)
- 403 "Player command failed: Premium required" - endpoint requires Spotify Premium
- 403 "Insufficient client scope" - add the missing scope and re-authorize
- 404 on `/me/player` - no active device (start playback on a device first)
- 404 on track/album - resource doesn't exist in the requested market

---

## Step 11: Market and locale

Always pass `market` to filter unavailable content and avoid duplicate results per country:

```python
# Use user's market from their account token
results = client.get("/search", q="radiohead", type="track", market="from_token")

# Or explicit market
results = client.get("/search", q="radiohead", type="track", market="US")

# locale for localized text (category/playlist names)
# format: {language}_{COUNTRY}
categories = client.get("/browse/categories", locale="es_MX")
```

---

## Tips

- **No recommendations for new apps**: The recommendations endpoint is gone for new apps. Use search with genre filters or curate playlists from top tracks instead.
- **Batch everything**: `GET /tracks?ids=id1,id2,...` (50 max) and `GET /albums?ids=...` (20 max) save significant API calls when processing collections.
- **Player needs an active device**: `PUT /me/player/play` returns 404 if no device is active. Always call `GET /me/player/devices` first and pass `device_id` if needed.
- **Playlist snapshot IDs**: Playlist modification responses include a `snapshot_id`. Store it if you need to track playlist versions, though it's not required for standard CRUD.
- **Dev Mode limits**: If users hit 403 on valid requests, check if the 5-user limit is reached in the Spotify dashboard.
- **Token storage**: Never log or expose access/refresh tokens. Store encrypted at rest. Refresh tokens should be treated like passwords.
- **Content not available**: If a track appears in search but fails to play, use the `restrictions` field in the track object to check why (`market`, `explicit`, `product`).
