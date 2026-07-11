
# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention used elsewhere (`add_to_collection`, `remove_from_collection`, `get_collection`). Updated the one call site in `routes/watchlist/watchlist.py` (both the import and the function call).
**How I verified:** Ran a project-wide search for `save_to_watchlist` in VS Code (Ctrl+Shift+F) to confirm no remaining references. Ran `pytest tests/ -v` — all 4 existing tests still pass.

## Comment 2 — Deduplication
**What I did:** Added a dedup check to `add_to_watchlist()` in `services/watchlist_service.py`, following the same pattern as `add_to_collection()` in `collection_service.py`. Before creating a new `WatchlistEntry`, the function now checks for an existing entry with the same `user_id` and `film_id` using `WatchlistEntry.query.filter_by(...).first()`. If found, it raises a new `AlreadyInWatchlistError` exception (mirroring `AlreadyInCollectionError`). Also updated `routes/watchlist/watchlist.py` to catch `AlreadyInWatchlistError` and return a 409 status, matching how the collection routes handle the same case.
**How I verified:** Ran `pytest tests/ -v` — all existing tests still pass. Manually confirmed the logic mirrors `add_to_collection()`'s dedup check line by line.

## Comment 3 — Missing test
**What I did:**
**How I verified:**

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->