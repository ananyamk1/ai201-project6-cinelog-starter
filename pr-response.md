# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI to orient in the codebase (summarizing `models.py`, `collection_service.py`, and `test_collection.py`) and to stress-test my Comment 4 and 5 arguments as a devil's advocate — I asked what counterargument a careful reviewer would raise and what tradeoff I wasn't acknowledging.

- **Comment 4:** The counterargument surfaced was that watchlists are lower-sensitivity than collections (future intent vs. viewing history), and that private-by-default can starve a young social platform of the public content its discovery features depend on. That was partly a gap in my draft, so I revised: I explicitly acknowledged the social-discovery tradeoff and added the "nudge users to opt in" middle path rather than treating public-default as simply wrong.
- **Comment 5:** The counterargument was alphabetical's stability/findability for large lists. I had already addressed the core of it (search/filter + a future `?sort=` param), so I kept my position but tightened the "browsed vs. looked-up" framing.

All final reasoning is my own, grounded in CineLog's context; AI was used to pressure-test, not to author the arguments.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, matching the `add_to_collection()` naming used in the collection service. Updated the one call site in `routes/watchlist/watchlist.py` — both the `import` line and the call inside the `add_film` route handler.

**How I verified:** I ran a project-wide search (`grep -rn "save_to_watchlist" --include='*.py'`) before and after the change. Before, it matched three places: the function definition, the import in the route, and the call in the route. After the rename, the same search returns zero matches, confirming no call site was missed. The full test suite (`pytest tests/ -v`) still passes (4 passed).

## Comment 2 — Deduplication
**What I did:** Added a deduplication check to `add_to_watchlist()`, following the same pattern as `add_to_collection()` in `services/collection_service.py`. After confirming the film exists, the function now queries `WatchlistEntry` for an existing row with the same `user_id`/`film_id`; if one exists it raises `AlreadyInWatchlistError` instead of inserting a duplicate. I added the new `AlreadyInWatchlistError` exception class to mirror the collection service's `AlreadyInCollectionError`.

**Understanding the model function:** In `add_to_collection()`, the check is `CollectionEntry.query.filter_by(user_id=user_id, film_id=film_id).first()`. `.first()` returns the existing entry object if a duplicate exists, or `None` if not. When a duplicate is detected, the function raises `AlreadyInCollectionError` and never reaches the insert — so no second row is written.

**How I verified:** I ran a manual check in an app context: added a film, then added the same `(user_id, film_id)` again. The second call raised `AlreadyInWatchlistError`, and a `WatchlistEntry.query...count()` confirmed exactly one row exists (the duplicate was not inserted). The full test suite still passes.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`. I used `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py` as my model and wrote the equivalent `test_add_to_watchlist_nonexistent_film_raises`. I copied the same fixture structure (`app` with an in-memory SQLite DB, `sample_user`, `sample_film`) and the same assertion structure: a `pytest.raises(FilmNotFoundError)` block around a call with a nonexistent film id.

**How I verified:** `pytest tests/test_watchlist.py -v` passes the new test, and `pytest tests/ -v` runs green across all 5 tests (4 collection + 1 watchlist). The test confirms `add_to_watchlist()` raises `FilmNotFoundError` for an unknown film id rather than a database integrity error.

## Comment 4 — Default visibility
**My position:** Change the default from `public=True` to `public=False` — watchlists should be private by default, with sharing an explicit opt-in.

**Reasoning:** A watchlist is a statement of *future intent* — films a user has decided they want to watch but hasn't yet. That is arguably more revealing than the collection (a record of what you've already watched): it can signal current mood, interests, or plans a user hasn't acted on. Defaulting `public=True` silently opts every new entry into being visible to others, which violates the principle of least surprise — a user adding a film to "save it for later" is not thinking "I am now broadcasting this." I'm optimizing for **user trust and control**: the safe default is the one that can't cause a privacy regret. A user who wants visibility can flip a toggle at any time; a user who didn't realize their watchlist was public can't un-share what was already seen. This is privacy-by-design applied to CineLog specifically.

**Tradeoff acknowledged:** `public=True` optimizes for the opposite goal — social discovery and network effects. CineLog is a social film-logging platform (Letterboxd-style), and public watchlists are exactly the content that powers feeds, "what your friends want to watch," and recommendation surfaces. Private-by-default starves those features of seed content and can make a young platform feel empty. A reasonable middle path that keeps my position: default to private, but actively *nudge* users to make watchlists public (an onboarding prompt or a one-tap "share your watchlist" CTA) — so discovery is driven by informed consent rather than an invisible default.

## Comment 4 — Default visibility (implementation status)
Per the milestone, Comment 4 is a written design conversation, so I have not changed the `public=True` default in code as part of this PR — the argument above is my recommendation for a follow-up. Flagging explicitly so the maintainer can decide whether to fold the default flip into this PR or track it separately.

## Comment 5 — Sort order
**My position:** I agree with the maintainer and switched `get_watchlist()` from alphabetical (`Film.title.asc()`) to date-added, newest first (`WatchlistEntry.date_added.desc()`).

**Reasoning:** A watchlist is a *backlog / queue*, not a reference index. The question a user asks it is "what do I want to watch next?" — and for that, recency of intent is the most relevant signal: the film you just added is the one that's top-of-mind. Newest-first surfaces it immediately instead of burying it under everything from "A". It also makes the two list views in CineLog behave consistently: `get_collection()` already sorts `date_added` descending, so a user learns one mental model ("most recent first") that applies to both their collection and their watchlist.

**Engagement with reviewer's point:** Alphabetical wasn't arbitrary — its strength is determinism and findability: the order is stable across reloads, and if you know a title you can scan to it. I don't think that outweighs recency for the *default*, because findability-by-title is better served by explicit search/filter than by the default sort, and a watchlist is browsed ("what's next") more than it's looked-up ("where is film X"). The right long-term answer is to make sort a query parameter (`?sort=title|date_added`) so alphabetical stays available for users who prefer it — but the sensible default is date-added, matching the maintainer's preference and the rest of the app.

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->