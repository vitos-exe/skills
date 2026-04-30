---
name: merge-scan
description: >
  Scans all audited playlists in library-state.json and identifies pairs that should
  be merged based on thematic and vibe similarity. Trigger when the user wants to
  consolidate their library, find redundant playlists, or reduce fragmentation across
  audited playlists. Requires at least 2 playlists with audit_status "done".
---

# Merge Scan

Analyzes the audited library fingerprint to surface playlists that belong together.
Discovery is entirely offline -- no API calls until the user confirms merges.

**Prerequisite**: `library-state.json` must exist with at least 2 playlists where
`audit_status` is `"done"`. Playlists with `pending`, `skipped`, or `merged` status
are ignored. Playlists with `action_taken: "merge"` (merged during a prior audit session)
are also excluded -- they no longer exist independently on Spotify.

---

## Step 1: Load library-state.json

Read `library-state.json` from the current working directory.

- **Missing**: stop -- run `spotify:library-audit` first.
- **Fewer than 2 eligible playlists**: stop -- not enough audited playlists to compare.
- **Proceed**: collect all playlists where `audit_status` is `"done"` and
  `action_taken` is not `"merge"`.

Build a fingerprint list:

```json
[
  {
    "id": "...",
    "name": "Playlist Name",
    "description": "<description_written>",
    "sample_tracks": ["Track - Artist", "..."]
  }
]
```

Report how many playlists are eligible for comparison.

---

## Step 2: Analyze for merge candidates

Using the fingerprints, reason about which pairs belong together. Think like a curator:

**Criteria for a merge candidate:**
- Same primary genre and subgenre
- Overlapping or identical listening context (late-night, workout, focus, etc.)
- Compatible emotional register and energy level
- Sample tracks feel like they could come from the same session or mixtape
- The two playlists would be stronger and more useful as one

**Criteria against merging:**
- Playlists cover different eras of the same genre (e.g. 80s post-punk vs 2000s post-punk revival)
- Similar genre but clearly different use cases (background study vs active listening)
- One playlist is a deliberate subset with its own distinct identity worth preserving
- The playlists are "similar to" each other in their descriptions but intentionally distinct

**Direction**: pick the target as the playlist with the stronger or more defined identity.
The smaller or more diffuse one absorbs into the larger one.

**Confidence levels:**
- `high`: nearly identical genre, vibe, and context -- obvious consolidation
- `medium`: significant overlap but some distinction worth noting
- `low`: plausible but uncertain -- surface it and let the user decide

Surface all candidate pairs regardless of confidence level.

---

## Step 3: Present candidates

Format as a ranked list, highest confidence first. Number each entry:

```
Merge candidates ({N} found across {M} playlists):

1. [HIGH] "{Playlist A}" into "{Playlist B}"
   {Curator prose explaining why these belong together}

2. [MEDIUM] "{Playlist C}" into "{Playlist D}"
   {Prose}

3. [LOW] ...
```

If no candidates are found, tell the user the library looks well-partitioned and exit.

Then ask:
> Which would you like to apply? Enter numbers (e.g. "1, 3"), "all", or "none".

---

## Step 4: Credentials (only if merges confirmed)

If the user approves at least one merge, use the `spotify-api` skill. Required scopes:

```
playlist-read-private playlist-read-collaborative playlist-modify-public playlist-modify-private
```

---

## Step 5: Apply confirmed merges

For each approved pair, use the `id` already present in the fingerprint built in
Step 1. These IDs point to the agent copy playlists -- do not use `original_id` for writes.
Do not rely on current Spotify display names, which may have been renamed during prior
audit sessions.

For each approved merge in order:

1. `GET /playlists/{source_id}/items` (paginated) -- collect all source track URIs
2. `POST /playlists/{target_id}/items` -- add in batches of 100
3. `PUT /playlists/{source_id}` -- rename to `[agent] [merged] {original_name}`

Use the inline `spotify()` helper from the `spotify-api` skill for all calls.

---

## Step 6: Update library-state.json

For each applied merge:

**Source playlist** (the one absorbed):
```json
{
  "audit_status": "merged",
  "action_taken": "merge",
  "merged_into": "Target Playlist Name",
  "original_id": "<unchanged -- leave as-is>",
  "sample_tracks": []
}
```

**Target playlist** (the one that absorbs): no status change -- remains `"done"`.
Merge the source's `sample_tracks` into the target's existing `sample_tracks`, deduplicating
by string. Cap at 15 total. Update `last_updated` to the current ISO timestamp.

---

## Step 7: Update SESSION.md

```markdown
# Session Log

## Last action
- Merge scan: applied {N} merge(s)
{list of "Playlist A" into "Playlist B" for each applied merge}

## Next step
- Run `spotify:merge-scan` again after more playlists are audited
- Or continue with `spotify:library-audit` to audit remaining pending playlists
```
