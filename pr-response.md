# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

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
**How I verified:**

## Comment 4 — Default visibility
> I notice watchlists default to `public=True`. We don't have a documented decision on default visibility for user lists. Before I can approve this, I need you to add a note to your PR description explaining your reasoning. I want to make sure we're being intentional here, not just inheriting a default.

**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
> I'd prefer watchlists to default to "date added" order rather than alphabetical. Most users want to see what they added recently. I'm open to discussion if you see it differently — but let's make a decision and document it.

**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
> A refactor merged to `main` that changed film IDs from integers to UUIDs. Your watchlist code still references integer IDs. Please rebase on `main` and update accordingly.

**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->