
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
**What I did:** Created `tests/test_watchlist.py`, following the fixture and structure pattern from `tests/test_collection.py`. Included the specifically requested test, `test_add_to_watchlist_nonexistent_film_raises` (modeled directly on `test_add_to_collection_nonexistent_film_raises`), plus two supporting tests: a happy-path test confirming a valid add creates a `WatchlistEntry`, and a duplicate test confirming the Comment 2 dedup fix actually works end to end.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` — all 3 tests pass. Ran the full suite with `pytest tests/ -v` to confirm nothing else broke.

## Comment 4 — Default visibility
**My position:** `public=True` should remain the default for watchlist entries.

**Reasoning:** CineLog is pitched as a community app, and a watchlist's community value only exists if it's visible. Features like friends seeing "you want to watch that too, let's do it," discovering films through what others plan to watch, and watch-party planning all depend on watchlists being seen by default. If the default were private instead, that's not "private until the user decides otherwise" in practice; most users never find or touch a visibility setting they don't know exists. The app would quietly degrade into a solo tracking tool for anyone who never discovered the toggle, silently turning off the exact social feature that differentiates a watchlist from a private to-do list.

**Tradeoff acknowledged:** This does expose users who never actively chose to be public, some of whom might prefer privacy by default. I accept this tradeoff because a watchlist represents intent ("I want to watch this"), which is lower-signal and less personal than a rated collection entry ("I watched this and rated it X"). Being public about *wanting* to watch a film carries less exposure risk than being public about a completed, rated viewing history, so the cost of the default being "wrong" for a given user is comparatively low.

## Comment 5 — Sort order

**Note:** While writing the sort-order test, discovered that `WatchlistEntry` had no `film` relationship defined on the `Film` model (unlike `CollectionEntry`, which has `backref="film"`). This was a pre-existing bug masked by the fact that no prior test exercised `get_watchlist()`. Fixed by adding `watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)` to `Film` in `models.py`.

**My position:** Sort by `date_added.asc()` (oldest first), rather than either the current alphabetical order or the reviewer's suggested `date_added.desc()` (newest first).

**Reasoning:** A watchlist is a "things I mean to get to" list, and the films most worth surfacing are the ones a user has been neglecting the longest, not the ones they just added (which are already fresh in their mind) and not whatever happens to come first alphabetically (which carries no meaningful signal for this feature). Sorting oldest-first turns the watchlist into a natural prompt to clear a backlog, which is the actual behavior a watchlist should encourage.

**Engagement with reviewer's point:** I agree with the reviewer's core instinct that time, not title, is the right sort axis, since alphabetical order doesn't reflect how anyone actually thinks about a watchlist. Where I differ is the direction: newest-first optimizes for confirming a recent action ("did my add work?"), while oldest-first optimizes for the actual purpose of the list (surfacing what's been sitting untouched). The honest tradeoff is that oldest-first is less intuitive at the moment of adding: a user adds a film and doesn't see it appear at the top, which can feel like the action didn't register.

## Comment 6 — Rebase
**What conflicted:** Running `git rebase origin/main` initially only flagged a conflict in `.gitignore` (both branches had independently added one; resolved by keeping my branch's version, which had the full ignore list). However, the real UUID migration issue did not surface as a git conflict at all — git's auto-merge silently dropped the entire `WatchlistEntry` class from `models.py` during replay, since `main`'s refactor commit rewrote large parts of that file and the line-based merge didn't detect an overlap warranting a conflict marker.

**How I resolved it:** After the rebase reported "Successfully rebased," I ran the test suite and manually inspected `models.py`, discovering `WatchlistEntry` was missing entirely despite being imported elsewhere. I re-added the class with `film_id` now typed as `db.String(36)` (UUID) instead of `db.Integer`, matching the new `Film.id` type from main's refactor. I also updated the `add_to_watchlist()` docstring to reflect that `film_id` is now a UUID string, not an integer.

**How I verified no conflict remains:** Ran `pytest tests/ -v` — all 8 tests pass. Ran `git diff origin/main feature/watchlist` across the watchlist-related files to manually confirm no other pieces were silently dropped during the auto-merge, since this incident showed that "no reported conflict" doesn't guarantee a correct merge.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->