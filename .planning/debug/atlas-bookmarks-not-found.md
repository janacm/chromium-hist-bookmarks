---
slug: atlas-bookmarks-not-found
status: resolved
trigger: alfred workflow shows "Browser History not found!" when searching ChatGPT Atlas bookmarks/history despite Atlas being checked in workflow config
created: 2026-05-12
updated: 2026-05-12
---

# Debug Session: atlas-bookmarks-not-found

## Symptoms

DATA_START
- **Expected:** Typing `bh <query>` (or `bb <query>`) in Alfred searches ChatGPT Atlas browser history/bookmarks and returns matching results.
- **Actual:** Alfred displays "Browser History not found! Ensure Browser is installed or choose available browser(s) in CONFIGURE WORKFLOW" even though Atlas is installed and the "ChatGPT Atlas" checkbox is enabled in the workflow config.
- **Error message:** "Browser History not found! Ensure Browser is installed or choose available browser(s) in CONFIGURE WORKFLOW" (purple Alfred result row).
- **Timeline:** New work — user is on branch `add-comet-browser-support` extending the workflow to support newer Chromium-based browsers (Comet was just added; Atlas support was the next attempt). Likely never worked.
- **Reproduction:** In Alfred prompt, type `bh vercel` with ChatGPT Atlas enabled in workflow config and ChatGPT Atlas app installed at `~/Library/Application Support/com.openai.atlas/`.
- **Config state:** Workflow config shows both `Comet` and `ChatGPT Atlas` checkboxes ticked.
DATA_END

## Current Focus

hypothesis: The ATLAS path constants in `src/chrom_bookmarks.py:28` and `src/chrom_history.py:29` point to `Library/Application Support/ChatGPT Atlas/Default/Bookmarks` and `.../History`, but ChatGPT Atlas actually stores its data at `Library/Application Support/com.openai.atlas/browser-data/host/{Profile}/{Bookmarks,History}`. The directory name differs ("com.openai.atlas" vs "ChatGPT Atlas") AND profiles are nested under an extra `browser-data/host/` path segment.
test: Run `ls "$HOME/Library/Application Support/com.openai.atlas/browser-data/host/Default/History"` and `ls "$HOME/Library/Application Support/ChatGPT Atlas/Default/History"` — only the first should exist.
expecting: First command succeeds (file exists), second fails (directory not found).
next_action: Verify the path discrepancy, then patch ATLAS entries in both bookmarks.py and history.py to use the correct path. Decide on profile handling (Default vs user-OXMa... UUID profiles — Atlas has multiple).

## Evidence

- timestamp: 2026-05-12 — `find "$HOME/Library/Application Support/com.openai.atlas" -name History -o -name Bookmarks` returned valid paths under `browser-data/host/Default/`, `browser-data/host/Guest Profile/`, and several `browser-data/host/user-OXMa3QbD0p7dYdCTwQHIvicF__*/` UUID-suffixed profiles.
- timestamp: 2026-05-12 — `ls "$HOME/Library/Application Support/" | rg atlas` shows `com.openai.atlas` directory exists, but no `ChatGPT Atlas` directory exists.
- timestamp: 2026-05-12 — `rg -in "atlas" src/` confirms two configured paths: `src/chrom_bookmarks.py:28` and `src/chrom_history.py:29`, both pointing to the non-existent `ChatGPT Atlas/Default/...`.
- timestamp: 2026-05-12 — Test from Current Focus passed: `ls "$HOME/Library/Application Support/com.openai.atlas/browser-data/host/Default/History"` returned the path; `ls "$HOME/Library/Application Support/ChatGPT Atlas/Default/History"` returned "No such file or directory" (exit 1). Hypothesis confirmed.
- timestamp: 2026-05-12 — Profile mtime survey: the active user profile is `user-OXMa3QbD0p7dYdCTwQHIvicF__eafceeb7-a585-4c3c-bd9c-e68836ce91ff` (most recently modified). `Default/History` exists but is essentially empty/stale (oldest mtime). `Default/Bookmarks` does NOT exist. Therefore a single-path fix to `.../Default/Bookmarks` would have found nothing and a single-path fix to `.../Default/History` would have returned an empty result set. Multi-profile expansion is required.
- timestamp: 2026-05-12 — Sanity check on the patched resolution: `glob.glob` over the patched Atlas pattern returns 3 Bookmarks files (Guest Profile + 2 user-UUID profiles) and 5 History files (Default + Guest Profile + 3 user-UUID profiles). The active user-UUID profile's Bookmarks file parses with the existing `get_json_from_file` and yields 4577 bookmark URL entries.

## Eliminated

- "Atlas reads chrome's BROWSER_MAP entry" — no; Atlas has its own `atlas` key in info.plist and its own dict entry.
- "Single-profile fix would suffice" — eliminated by profile mtime + presence survey above. `Default` is stale and lacks Bookmarks; real data lives in `user-<UUID>__<UUID>` directories.

## Resolution

root_cause: The `atlas` entries in `BOOKMARKS_MAP` (src/chrom_bookmarks.py:28) and `HISTORY_MAP` (src/chrom_history.py:29) pointed to `Library/Application Support/ChatGPT Atlas/Default/{Bookmarks,History}`, which does not exist. ChatGPT Atlas (bundle id `com.openai.atlas`) stores data at `Library/Application Support/com.openai.atlas/browser-data/host/<profile>/{Bookmarks,History}`, with an additional `browser-data/host/` path segment and multiple per-user profile directories (`Default`, `Guest Profile`, and several `user-<UUID>__<UUID>` directories). The previous single-path-per-browser pattern could not address the multi-profile layout even with the right base directory, because the active user profile lives under a non-`Default` UUID directory.
fix: Two coordinated changes, applied without special-casing Atlas:
  1. Generalized `paths_to_bookmarks()` (src/chrom_bookmarks.py) and `history_paths()` (src/chrom_history.py) to expand any map entry containing shell-style wildcards (`*`, `?`, `[`) via `glob.glob` before applying the `os.path.isfile` filter. Non-glob entries take a one-element-list fast path, preserving the exact existing behavior for every other browser. Both files now `import glob`.
  2. Updated the `atlas` entries to `Library/Application Support/com.openai.atlas/browser-data/host/*/Bookmarks` and `.../*/History`, so every Atlas profile (Default, Guest, and user-UUID) is picked up automatically. Downstream deduplication (`removeDuplicates` in both files) already collapses overlapping entries across profiles, so the fan-out is safe.
verification:
  - `python3 -m py_compile src/chrom_bookmarks.py src/chrom_history.py` — passes.
  - Filesystem simulation of patched `paths_to_bookmarks()` returns 3 real Atlas Bookmarks files (active user profile included).
  - Filesystem simulation of patched `history_paths()` returns 5 real Atlas History files.
  - The active user profile's Bookmarks file parses with the existing `get_json_from_file` and produces 4577 bookmark URL entries — confirming Atlas uses the standard Chromium Bookmarks JSON schema, no parser change needed.
  - Atlas History files are standard Chromium SQLite DBs (`urls`/`visits` tables); the existing `sql()` query in chrom_history.py applies unchanged.
files_changed:
  - src/chrom_bookmarks.py — added `import glob`; rewrote `paths_to_bookmarks()` to expand glob entries; updated `BOOKMARKS_MAP["atlas"]` to the `com.openai.atlas/browser-data/host/*/Bookmarks` pattern; added comment documenting the wildcard convention.
  - src/chrom_history.py — added `import glob`; rewrote `history_paths()` to expand glob entries; updated `HISTORY_MAP["atlas"]` to the `com.openai.atlas/browser-data/host/*/History` pattern; added comment documenting the wildcard convention.
