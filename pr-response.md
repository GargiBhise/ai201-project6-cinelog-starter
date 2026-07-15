# PR Response Doc — CineLog Watchlist Feature

## AI Usage
AI Usage

I used Claude for orientation on the existing codebase patterns, debugging a Flask/SQLAlchemy circular-import error, reviewing my own code drafts against existing conventions (e.g. catching a variable-naming bug and wrong exception type in my Comment 2 draft), stress-testing my Comment 4/5 arguments as a devil's advocate (which surfaced the CollectionEntry visibility inconsistency I hadn't considered), and step-by-step guidance through interactive rebase, including recovering from a silent models.py conflict that dropped WatchlistEntry. The reasoning in Comments 4–5 and all code/tests are my own; AI was used to verify and unblock, not to generate arguments or implementation.

## Comment 1 — Rename
> `save_to_watchlist()` should follow the project's naming convention. Compare with `add_to_collection()` — the pattern here is `verb_to_noun`. Please rename to `add_to_watchlist()` and update all call sites.

**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to follow the project's `verb_to_noun` naming convention. I updated the import and function call in `routes/watchlist/watchlist.py`. I also used `git grep -n "save_to_watchlist"` and `git grep -n "add_to_watchlist"` to confirm all code references were updated.

**How I verified:**
I ran `git diff` to review the exact changes in both files and confirm that only the function name, import, and call site were changed. I then ran `pytest tests/ -v` and confirmed that all existing tests passed.

## Comment 2 — Deduplication
> What happens if a user calls this with a film that's already on their watchlist? The current implementation would add a duplicate entry. Please handle this case.

**What I did:**
Added a duplicate check in `add_to_watchlist()` using the same `user_id` and `film_id` lookup pattern as `add_to_collection()`. If a matching `WatchlistEntry` already exists, the function now raises `AlreadyOnWatchlistError` instead of creating another row.

**How I verified:**
I compared the implementation with the deduplication logic in `add_to_collection()`, reviewed the exact changes using `git diff`, and ran `pytest tests/ -v`. All existing tests passed.

## Comment 3 — Missing test
> Please add a test for the case where `film_id` doesn't exist in the database. Look at the existing tests in `test_collection.py` — the pattern is there.
**What I did:**
Created `tests/test_watchlist.py` and added
`test_add_to_watchlist_nonexistent_film_raises()`, following the fixture
setup and `pytest.raises()` structure used by
`test_add_to_collection_nonexistent_film_raises()` in
`tests/test_collection.py`.

I also added `test_add_to_watchlist_duplicate_raises()` to directly verify
the deduplication behavior added in Comment 2. The test confirms that the
second add raises `AlreadyOnWatchlistError` and that only one matching
`WatchlistEntry` remains in the database.

**How I verified:**
I ran `pytest tests/test_watchlist.py -v` to verify the new watchlist tests,
then ran `pytest tests/ -v` to confirm the entire test suite passed. All 6
tests passed.

## Comment 4 — Default visibility
> I notice watchlists default to `public=True`. We don't have a documented decision on default visibility for user lists. Before I can approve this, I need you to add a note to your PR description explaining your reasoning. I want to make sure we're being intentional here, not just inheriting a default.

**My position:**
I would change the default to `public=False`.

**Reasoning:**
A watchlist represents a user's future viewing interests and may include personal, niche, or sensitive topics. Even if CineLog does not yet have a public discovery feed, the presence of a `public` field suggests that watchlists may be exposed to other users now or in future features. I do not think the application should assume that every user is comfortable sharing that information. A private default gives users control first and allows them to explicitly make the watchlist public when they choose.

Unlike `CollectionEntry`, which has no visibility control, a watchlist may contain exploratory choices a user is only considering and has not yet committed to. That makes the watchlist reasonable to treat more cautiously, although the inconsistency also suggests that CineLog should eventually define a broader visibility policy for both kinds of lists.

**Tradeoff acknowledged:**
A `public=True` default could improve discovery and make CineLog feel more social because more watchlists would be immediately available for browsing. Defaulting to private may reduce participation in those social features because some users may never opt in. I still prefer explicit consent because accidental sharing is harder to reverse than asking an interested user to enable visibility.

## Comment 5 — Sort order
> I'd prefer watchlists to default to "date added" order rather than alphabetical. Most users want to see what they added recently. I'm open to discussion if you see it differently — but let's make a decision and document it.

**My position:**
I agree with changing the default sort order to date added, newest first.

**Reasoning:**
A watchlist reflects films a user intends to watch, and recent additions are likely to match the user's current interests. Showing those first makes it easier to revisit something recently discovered without remembering the exact title. It also keeps the watchlist consistent with `get_collection()`, which already uses newest-first ordering.

**Engagement with reviewer's point:**
I agree that recent additions are likely to be the most useful default. Alphabetical order is still better when a user knows the exact title, especially in a large watchlist, so a future improvement could allow users to choose between alphabetical and recently added sorting.

## Comment 6 — Rebase
> A refactor merged to `main` that changed film IDs from integers to UUIDs. Your watchlist code still references integer IDs. Please rebase on `main` and update accordingly.

**What conflicted:**
The rebase first produced an add/add conflict in `.gitignore` because both `main` and my feature branch had added that file. After the rebase completed, the test suite revealed that `WatchlistEntry` was missing from `models.py`, because `main`'s UUID refactor had replaced the earlier model definition.

**How I resolved it:**
I combined the `.gitignore` rules, removed the conflict markers, staged the file, and continued the rebase. I then restored `WatchlistEntry` in `models.py` using UUID string columns for `id`, `user_id`, and `film_id`, and restored the relationships from `User` and `Film`. I also updated the watchlist route and service docstrings so `film_id` is documented as a UUID rather than an integer.

**How I verified no conflict remains:**
I searched the codebase for remaining integer-based `film_id` references, ran `pytest tests/ -v`, and confirmed that all 6 tests passed. I also ran `git log --merges origin/main..HEAD`, which returned no output, confirming that no merge commits remain.

## PR Description
Overview

This PR adds a watchlist feature to CineLog, allowing users to save films they intend to watch (as distinct from CollectionEntry, which tracks films already watched). It includes:


A new WatchlistEntry model (user_id, film_id, date_added, public)
Service functions in services/watchlist_service.py: add_to_watchlist() and get_watchlist()
REST endpoints in routes/watchlist/watchlist.py: GET /watchlist/<user_id> and POST /watchlist/<user_id>/add
Test coverage in tests/test_watchlist.py for both the nonexistent-film and duplicate-entry cases


Design decisions


Default visibility (public): Changed to default False. Watchlists can reveal personal or exploratory interests before a user has committed to them, and a private default puts the user in control of when to share that. See Comment 4 for full reasoning.
Sort order: Watchlists are sorted by date_added descending (newest first), matching get_collection()'s existing sort behavior and prioritizing recent additions over alphabetical browsing. See Comment 5 for full reasoning.
Deduplication: add_to_watchlist() raises AlreadyOnWatchlistError if the film is already on the user's watchlist, mirroring add_to_collection()'s existing pattern for AlreadyInCollectionError.


Manual testing steps


Start the app: flask run (with FLASK_APP=app:create_app set), or python app.py if not using the app-factory import pattern.
Create a user and film via the existing /collection or /films endpoints, or seed data directly.
Add a film to a user's watchlist:


   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<uuid>"}'


Confirm it appears, sorted newest-first:


   curl http://127.0.0.1:5000/watchlist/<user_id>


Repeat step 3 with the same user_id/film_id and confirm a 409-style error is returned instead of a duplicate entry.
Run the full test suite: pytest tests/ -v (expect 6 passed).



Screenshot of git log --oneline : commitHistory.jpg