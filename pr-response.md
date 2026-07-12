# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, matching the `add_to_collection()` naming used in the collection service. Updated the one call site in `routes/watchlist/watchlist.py` — both the `import` line and the call inside the `add_film` route handler.

**How I verified:** I ran a project-wide search (`grep -rn "save_to_watchlist" --include='*.py'`) before and after the change. Before, it matched three places: the function definition, the import in the route, and the call in the route. After the rename, the same search returns zero matches, confirming no call site was missed. The full test suite (`pytest tests/ -v`) still passes (4 passed).

## Comment 2 — Deduplication
**What I did:** Added a deduplication check to `add_to_watchlist()`, following the same pattern as `add_to_collection()` in `services/collection_service.py`. After confirming the film exists, the function now queries `WatchlistEntry` for an existing row with the same `user_id`/`film_id`; if one exists it raises `AlreadyInWatchlistError` instead of inserting a duplicate. I added the new `AlreadyInWatchlistError` exception class to mirror the collection service's `AlreadyInCollectionError`.

**Understanding the model function:** In `add_to_collection()`, the check is `CollectionEntry.query.filter_by(user_id=user_id, film_id=film_id).first()`. `.first()` returns the existing entry object if a duplicate exists, or `None` if not. When a duplicate is detected, the function raises `AlreadyInCollectionError` and never reaches the insert — so no second row is written.

**How I verified:** I ran a manual check in an app context: added a film, then added the same `(user_id, film_id)` again. The second call raised `AlreadyInWatchlistError`, and a `WatchlistEntry.query...count()` confirmed exactly one row exists (the duplicate was not inserted). The full test suite still passes.

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