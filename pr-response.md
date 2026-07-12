# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI to orient in the codebase (summarizing `models.py`, `collection_service.py`, and `test_collection.py`) and to stress-test my Comment 4 and 5 arguments as a devil's advocate — I asked what counterargument a careful reviewer would raise and what tradeoff I wasn't acknowledging.

- **Comment 4:** The counterargument surfaced was that watchlists are lower-sensitivity than collections (future intent vs. viewing history), and that private-by-default can starve a young social platform of the public content its discovery features depend on. That was partly a gap in my draft, so I revised: I explicitly acknowledged the social-discovery tradeoff and added the "nudge users to opt in" middle path rather than treating public-default as simply wrong.
- **Comment 5:** The counterargument was alphabetical's stability/findability for large lists. I had already addressed the core of it (search/filter + a future `?sort=` param), so I kept my position but tightened the "browsed vs. looked-up" framing.

I also used AI as a final check on my commit history — I pasted `git log --oneline` and asked whether the messages followed conventional-commit format and whether any commit bundled multiple logical changes. I then verified the messages myself against the Conventional Commits spec before finalizing.

All final reasoning is my own, grounded in CineLog's context; AI was used to pressure-test, not to author the arguments.

## Commit history (`git log --oneline origin/main..HEAD`)
```
docs: document stretch features in pr-response
test: add tests for remove_from_watchlist and visibility toggle
feat: add watchlist visibility toggle endpoint
feat: add remove_from_watchlist endpoint
docs: finalize pr-response with PR description and commit-history log
fix: restore WatchlistEntry film_id as UUID after main rebase
feat: sort watchlist by date added instead of alphabetically
test: add test for nonexistent film in add_to_watchlist
fix: add deduplication check to prevent duplicate watchlist entries
fix: rename save_to_watchlist to add_to_watchlist per naming convention
fix: use db.session.get for film retrieval in collection and watchlist services
feat: add watchlist model and add_to_watchlist endpoint
```
All conventional commits (`feat:`/`fix:`/`test:`/`docs:`), each one logical change, linear on top of `main` with no merge commits. (The required six-comment work is the bottom seven commits; the top four add the stretch features and this documentation.)

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
**What conflicted:** I ran `git fetch origin` and `git rebase origin/main`. Two things had to be reconciled:
1. **`pr-response.md` (add/add conflict):** `main` had added an empty template of this file and my branch had added a filled-in version, so git couldn't auto-merge. This was a textual conflict with `<<<<<<<`/`>>>>>>>` markers.
2. **The UUID migration (`models.py` + watchlist code):** `main` migrated film IDs from integer to UUID — `Film.id` and `CollectionEntry.film_id` became `db.String(36)`. My branch was written against the old integer schema. This was the substantive conflict the review comment was about. Notably, git did *not* leave conflict markers for it: main's refactor commit had deleted the region after `CollectionEntry`, so the 3-way merge silently dropped my `WatchlistEntry` class (which lived there) and kept my integer-based watchlist code — a semantic conflict, not a textual one.

**How I resolved it:**
- For `pr-response.md`, I took my branch's version (`git checkout --theirs`), since the later commits re-apply their own sections on top during the replay.
- For the UUID migration, I re-added the `WatchlistEntry` model that the merge had dropped, this time with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"))` to match the post-refactor `CollectionEntry`. I also updated the now-stale integer references in the watchlist docstrings (`services/watchlist_service.py` "film_id (int)" → "film_id (str): UUID"; the route body doc `<int>` → `<str UUID>`). The service already used `db.session.get(Film, film_id)`, which works unchanged with UUID string keys.

**How I verified no conflict remains:**
- `grep -n "WatchlistEntry" models.py` confirms the model is back with a `String(36)` `film_id`, and no `db.Integer` film-id references remain in the watchlist code.
- `pytest tests/ -v` passes all 5 tests, and I ran a manual end-to-end check: created a `Film` (whose `id` is now a UUID string), added it via `add_to_watchlist()`, confirmed the stored `film_id` matched the UUID, and confirmed a duplicate add raises `AlreadyInWatchlistError`.
- `git log --merges origin/main..HEAD` returns nothing and `git log --oneline origin/main..HEAD` shows a linear stack of my six commits on top of `origin/main` — confirming the rebase produced no merge commits.

## PR Description

### What this feature does
Adds a **watchlist** to CineLog — a per-user list of films a user wants to watch later (distinct from the collection, which is films already watched). It adds a `WatchlistEntry` model and two endpoints:
- `POST /watchlist/<user_id>/add` with body `{ "film_id": "<uuid>" }` — adds a film to the user's watchlist.
- `GET /watchlist/<user_id>` — returns the user's watchlist.

`add_to_watchlist()` validates that the film exists (raising `FilmNotFoundError` otherwise) and de-duplicates: adding the same film twice raises `AlreadyInWatchlistError` instead of creating a second row.

### Design decisions
1. **Default visibility — recommend private (`public=False`).** A watchlist signals future intent and is arguably more revealing than viewing history, so the safe default is one that can't cause a privacy regret; sharing should be an explicit opt-in. I acknowledge the tradeoff: public-by-default fuels social discovery/feeds, which matters for a social film platform — so the recommended middle path is private-by-default plus an explicit "share your watchlist" nudge. (The code currently ships the original `public=True`; the default flip is flagged as a follow-up in Comment 4.)
2. **Sort order — date added, newest first.** A watchlist is a backlog answering "what do I want to watch next?", so recency of intent beats alphabetical, and it matches `get_collection()`'s ordering for a consistent mental model. Alphabetical's findability is better served by explicit search/a future `?sort=` param than by the default sort.

### How to manually test
1. Install deps and run the app: `pip install -r requirements.txt` then `flask run` (or `python app.py`).
2. Create a user and a film (via the existing endpoints / a DB seed) and note their UUIDs.
3. **Add:** `POST /watchlist/<user_id>/add` with `{ "film_id": "<film_uuid>" }` → expect `201` and the entry JSON.
4. **Dedup:** repeat the same POST → expect an error (film already on watchlist), and the watchlist still contains one copy.
5. **Not found:** POST with a random UUID that isn't a real film → expect a "film not found" error, not a DB integrity error.
6. **View / sort:** add a second film, then `GET /watchlist/<user_id>` → expect both films with the most-recently-added first.
7. **Automated:** `pytest tests/ -v` → all tests pass (includes `test_add_to_watchlist_nonexistent_film_raises`).

## Stretch features

### Stretch 1 — `remove_from_watchlist()`
**What I did:** Added `remove_from_watchlist(user_id, film_id)` to `services/watchlist_service.py`, following the existing `remove_from_collection()` pattern in the collection service. It looks up the `WatchlistEntry` by `user_id`/`film_id`; **if the film isn't on the watchlist it raises `NotInWatchlistError`** (a new exception mirroring `NotInCollectionError`) rather than silently succeeding; otherwise it deletes the entry and returns `True`. Exposed via `DELETE /watchlist/<user_id>/remove` (body `{ "film_id": "<uuid>" }`), which maps the error to a `404`, matching the collection route. **Test:** `test_remove_from_watchlist_not_present_raises` in `tests/test_watchlist.py` asserts the not-present case raises `NotInWatchlistError`. Commit: `feat: add remove_from_watchlist endpoint`.

### Stretch 2 — Second test
Beyond the Comment 3 test, I added two more tests in the `test: add tests for remove_from_watchlist and visibility toggle` commit. The notable added edge case is `test_remove_from_watchlist_not_present_raises`: I chose the "remove something that was never added" case because it's the boundary where a naive implementation would silently no-op (or throw a low-level error) instead of giving the caller a clear domain error — exactly the failure mode `NotInWatchlistError` exists to prevent. The second added test, `test_set_watchlist_visibility_updates_flag`, covers the toggle below.

### Stretch 3 — Visibility toggle endpoint
**What I did:** Added `set_watchlist_visibility(user_id, film_id, public)` and exposed it via `PATCH /watchlist/<user_id>/visibility` (body `{ "film_id": "<uuid>", "public": true|false }`). **How `public` works:** each `WatchlistEntry` has a boolean `public` column that **defaults to `True`** (an entry is public when created); this endpoint lets a caller flip it — e.g. `PATCH` with `{"public": false}` makes that entry private, `{"public": true}` makes it public again. It raises `NotInWatchlistError` (→ `404`) if the film isn't on the user's watchlist. This is the concrete "user can flip a toggle at any time" mechanism referenced in my Comment 4 argument. **Test:** `test_set_watchlist_visibility_updates_flag` confirms an entry starts `public=True` and becomes `public=False` after the call. Commit: `feat: add watchlist visibility toggle endpoint`.