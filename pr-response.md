# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used AI for codebase orientation at the start of the project: I asked it to summarize `models.py`, `services/collection_service.py`, and `tests/test_collection.py` so I could identify the naming and testing patterns before editing watchlist code. I verified each summary against the actual files before making changes.

## Comment 1 — Rename

**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` and updated the route import/call site in `routes/watchlist/watchlist.py` to match the project's `verb_to_noun` naming pattern.
**How I verified:** Ran a workspace search for `save_to_watchlist` / `add_to_watchlist` to confirm there were no stale call sites, then validated the edited Python files with the test suite after the rename landed.

## Comment 2 — Deduplication

**What I did:** Added a duplicate check in `add_to_watchlist()` using `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` and raised `AlreadyInWatchlistError` when the same film was already saved for that user. I also returned a `409` from the route so the API handles the conflict explicitly.
**How I verified:** Added a duplicate-path test in `tests/test_watchlist.py` that creates one entry and confirms the second add raises `AlreadyInWatchlistError`, then confirmed the test passed and the route/service files had no syntax errors.

## Comment 3 — Missing test

**What I did:** Created `tests/test_watchlist.py` following the same fixture/assertion pattern as `tests/test_collection.py`, including the nonexistent-film case requested in review. I used an integer `film_id` in the test because this branch still had the pre-refactor integer film model at the time I wrote it.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` and confirmed the nonexistent-film test passed along with the happy-path and duplicate-path checks in the new file.

## Comment 4 — Default visibility

**My position:** Keep `public=True` as the default for new watchlist entries.
**Reasoning:** CineLog is meant to support a community film-tracking workflow, so the default should optimize for discoverability and low friction rather than requiring every user to opt in to visibility before the feature is useful. A public-by-default watchlist makes it easier for friends and followers to see what someone is planning to watch, which fits the app's social context better than a hidden-by-default list. It also keeps the common case simple in the API and model layer.
**Tradeoff acknowledged:** This choice favors sharing over privacy, so it can expose a user's watchlist if they expected a personal default. That risk is real, which is why the visibility behavior needs to be clearly documented and why an explicit `public` override is valuable for callers who want a different setting.

## Comment 5 — Sort order

**My position:** Keep alphabetical sorting for the current watchlist response.
**Reasoning:** Alphabetical order makes the watchlist easy to scan when a user comes back later and wants to find a specific title quickly. The watchlist is a "to watch" inventory more than a timeline, and `date_added` is already stored on each entry if a future UI wants recency-based sorting or filtering. Alphabetical ordering is also stable: the position of existing films does not keep shifting as new items are added.
**Engagement with reviewer's point:** I understand why `date_added` is appealing, because it highlights the most recently added films and matches the emotional "what did I just save?" use case. I still think alphabetical is a better default for the current API because it prioritizes lookup over recency, and the stored metadata leaves room to add a recency view later without losing information.

## Comment 6 — Rebase

**What conflicted:** `models.py` conflicted because `main` had already migrated `Film.id` to UUIDs while my watchlist branch still added `WatchlistEntry` with an integer `film_id`.
**How I resolved it:** I kept the new `WatchlistEntry` model, changed its `film_id` column to `db.String(36)` to match the UUID-based `Film` model, and updated the watchlist service, route docstring, and nonexistent-film test to use UUID-aligned wording and values.
**How I verified no conflict remains:** The rebase completed successfully, `pytest tests/test_watchlist.py -v` passed after the UUID cleanup, and I checked that the rebased branch range has no merge commits.

## PR Description

This watchlist feature lets a user save films they want to watch later and retrieve that list through the API. It adds a watchlist model, a service layer for adding and listing watchlist entries, and REST endpoints for viewing a user's watchlist and adding a film to it.

I kept `public=True` as the default visibility so watchlists fit CineLog's social discovery workflow, and I kept the response sorted alphabetically so users can scan the list predictably when they come back later. Both choices favor simple everyday use while still preserving the data needed for future visibility or recency-based options.

Manual test steps:
1. Start the app with `python app.py`.
2. Create or use an existing user and a valid film UUID in the database.
3. Send `POST /watchlist/<user_id>/add` with `{"film_id": "<uuid>"}` and confirm the API returns `201`.
4. Send `GET /watchlist/<user_id>` and confirm the film appears in the list.
5. Repeat the same add request and confirm the API returns `409` for a duplicate.
6. Try a nonexistent film UUID and confirm the API returns `404`.

Git log screenshot: [git-log-snapshot.png](git-log-snapshot.png)
