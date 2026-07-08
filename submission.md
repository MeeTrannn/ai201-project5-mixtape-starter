# AI Usage

I used Cursor's AI assistant throughout this project as a collaborator — not as a substitute for reading the code myself. Here's an honest breakdown of how that worked.

## What I asked the AI to explain or trace

- **`update_listening_streak`** — Before touching Issue #1, I asked the AI to walk through the streak logic line by line. It explained the calendar-day rules and flagged the suspicious `today.weekday() != 6` guard that didn't match the docstring. That pointed me to the right branch without me having to guess.
- **`get_friends_listening_now`** — I asked for a full explanation of the feed query, the 24-hour cutoff, and the deduplication loop. This helped me understand why Nova's bug (yesterday evening still showing at 9am) was a time-window problem, not a deduplication problem.
- **`search_songs`** — I asked the AI to explain the query, especially why `.outerjoin(song_tags, ...)` was there. It explained that joins multiply rows in SQL and that tags were already available through `song.to_dict()` → `self.tags`, so the join looked unnecessary.
- **Follow-up on the join** — I specifically asked *why* the original author might have added `outerjoin` and *why* 3 tags produce 3 rows. The AI drew out the table-by-table join behavior, which finally made the duplicate-search bug click for me.

## What the AI helped me understand

- Python's `datetime.weekday()` numbering (Monday = 0, Sunday = 6) and how a wrong weekday check could block streak increments on Sundays.
- The difference between a **rolling 24-hour window** and a **calendar-day boundary** — critical for the feed bug.
- How SQL `JOIN` / `OUTER JOIN` on a many-to-many association table creates one result row per related record, which is why tag count correlated with duplicate count in Simone's report.
- The project's call-chain pattern: route → service → model (e.g. `routes/songs.py` → `streak_service.py` → `models.py`).

## Where I had to verify things myself (or the AI was incomplete)

- **Issue #1 — my first fix idea was wrong.** I asked whether changing `weekday() != 6` to `weekday() != 5` would fix the streak bug. The AI correctly pushed back: that would just move the bug to Saturdays. I wouldn't have caught that swap on my own without thinking through what `weekday() == 5` actually means. The real fix was removing the weekday check entirely.
- **Issue #1 — I still ran the tests myself.** The AI ran `pytest tests/test_streaks.py` and confirmed `test_streak_increments_on_sunday` failed before the fix and passed after. I treated that as the ground truth, not the explanation alone.
- **Issue #2 — test setup wasn't automatic.** When the AI first wrote feed tests, they failed on a foreign-key constraint because `song.shared_by` referenced a user ID before the user was committed. The AI caught and fixed that, but it reminded me that generated test code still needs the same database ordering rules as anything I'd write by hand.
- **Issue #3 — the AI's first explanation was only half the story.** The existing `test_search_no_duplicates_multi_tag_song` test actually **passed even with the buggy `outerjoin` still in place**. The AI had to dig deeper and run raw SQL to show the join really does return 3 rows for a 3-tag song — SQLAlchemy's legacy `Query.all()` was deduplicating `Song` objects in Python, which masked the bug at the ORM layer. I wouldn't have trusted "remove the join" based on the passing test alone; the raw SQL row count (3 rows → 1 row after fix) is what convinced me the root cause was real.
- **Submissions** — The AI drafted the structured write-ups for each issue (reproduction, root cause, fix, side-effect check). I reviewed them against what I actually observed in the code and tests and adjusted wording where needed (e.g. being precise about `weekday()` values, documenting the SQL-vs-ORM discrepancy for Issue #3).

## Overall

The AI was most useful for **explaining unfamiliar code quickly**, **tracing routes to services**, and **sanity-checking my fix ideas** (especially when I almost applied the wrong weekday). I still had to read the actual files, run pytest, and in one case go past a misleading passing test to confirm the search bug at the SQL level. The fixes are small, but understanding *why* they work required both the AI's explanations and my own verification.

---

# Issue #1 — My listening streak keeps resetting

## How you reproduced it

1. Read Kenji's report: streak was 12 on Saturday night, reset to 1 on Sunday morning despite listening both days.
2. Opened `services/streak_service.py` and traced `update_listening_streak`, the function called whenever a user listens to a song.
3. Ran the existing test suite before changing any code:
   ```bash
   python -m pytest tests/test_streaks.py -v
   ```
4. Four tests passed; `test_streak_increments_on_sunday` failed:
   - Simulated listening on Saturday 2024-06-15 (`weekday() == 5`), streak set to 1.
   - Simulated listening on Sunday 2024-06-16 (`weekday() == 6`), one calendar day later.
   - Expected streak 2; actual streak 1 — matching Kenji's report exactly.

## How you found the root cause

1. Started from the bug report's endpoint (`GET /users/<id>/streak`) and followed it to `routes/users.py` → `get_streak()` in `services/streak_service.py`.
2. Traced the write path: `routes/songs.py` → `record_listening_event()` → `update_listening_streak()`.
3. Read the `elif` branch at line 73. The docstring says "if the user listened yesterday, streak increments by 1," but the code had an extra guard: `and today.weekday() != 6`.
4. Confirmed with Python's docs: `datetime.weekday()` returns 6 for Sunday. So on Sunday, even when `days_since_last == 1` (listened yesterday on Saturday), the condition evaluated to `False` and execution fell through to the `else` branch, which resets the streak to 1.
5. The failing Sunday test made this the specific cause — not a vague "streak logic" problem, but one exact comparison blocking the increment path on Sundays.

## The root cause

In `update_listening_streak`, the streak-increment branch required `days_since_last == 1 and today.weekday() != 6`. Python's `weekday()` returns 6 for Sunday. That means any listen on a Sunday where the previous listen was exactly one calendar day ago (e.g. Saturday → Sunday) skipped the increment branch and hit the reset branch (`listening_streak = 1`) instead. The weekday check had no basis in the documented streak rules and incorrectly treated consecutive Saturday-to-Sunday listens as a broken streak.

## Your fix and side-effect check

**Change:** Removed `and today.weekday() != 6` from the increment condition, leaving `elif days_since_last == 1:`.

**Why this fixes it:** Streak updates now depend only on calendar-day gaps, which matches the docstring and product behavior. Saturday → Sunday is one day apart, so the streak increments. Skipping a day (`days_since_last >= 2`) still resets to 1.

**Why not `weekday() != 5`:** That would unblock Sunday but block Saturday — `weekday() == 5` is Saturday, so the same bug would appear every Saturday instead. The weekday guard should not exist at all; streaks are day-based, not week-based.

**Side-effect check:** Re-ran the full streak test suite after the fix:
```bash
python -m pytest tests/test_streaks.py -v
```
All 5 tests pass:
- New user starts at 1
- Consecutive days increment (Mon → Tue)
- Same-day double listen does not double-count
- Skipped day resets to 1
- Saturday → Sunday increments (the previously failing case)

---

# Issue #2 — Friends Listening Now shows people from yesterday

## How you reproduced it

1. Read Nova's report: at 9am she saw darius "listening now" to a song he played at 11pm the previous night (~10 hours ago).
2. Traced the endpoint `GET /feed/<user_id>/listening-now` in `routes/feed.py` to `get_friends_listening_now()` in `services/feed_service.py`.
3. Wrote a test mirroring Nova's scenario before changing code (`tests/test_feed.py`):
   - Nova and darius are friends.
   - Darius listened at 11pm on June 16.
   - Nova checks the feed at 9am on June 17.
4. With the original 24-hour rolling window, darius would still be within range (only 10 hours ago). The new test `test_listening_now_excludes_friend_who_listened_yesterday` encodes the expected behavior: feed should be empty.

## How you found the root cause

1. Started at the route (`routes/feed.py` → `listening_now`) and followed the call into `services/feed_service.py`.
2. Found `RECENT_THRESHOLD = timedelta(hours=24)` and `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`.
3. The query filters with `ListeningEvent.listened_at >= cutoff` — a **rolling 24-hour window**, not a calendar-day boundary.
4. Nova's scenario confirms it: 11pm yesterday to 9am today is only 10 hours, so darius's event passes the filter even though it happened on a different calendar day. The product expects "today only," but the code measures elapsed time.

## The root cause

`get_friends_listening_now` used `datetime.now(timezone.utc) - timedelta(hours=24)` as its cutoff. That includes any listen from the past 24 hours regardless of calendar date. A friend who listened at 11pm yesterday is still within that window at 9am the next morning, so they incorrectly appear in "Friends Listening Now." The feed was time-based (hours elapsed) when it should be date-based (listened today).

## Your fix and side-effect check

**Change:** Replaced the rolling 24-hour cutoff with the start of the current UTC calendar day:

```python
now = datetime.now(timezone.utc)
start_of_today = now.replace(hour=0, minute=0, second=0, microsecond=0)
# ...
ListeningEvent.listened_at >= start_of_today
```

Removed the unused `RECENT_THRESHOLD` constant.

**Why this fixes it:** Any event before midnight UTC is excluded once a new day begins. Darius listening at 11pm June 16 no longer appears when Nova checks at 9am June 17. Friends who listened earlier the same day still show up.

**Side-effect check:** Ran feed and streak tests after the fix:
```bash
python -m pytest tests/test_feed.py tests/test_streaks.py -v
```
All 7 tests pass:
- Friend who listened earlier today → included
- Friend who only listened yesterday evening → excluded (Nova's bug)
- Streak logic unchanged (5 existing tests still pass)

`get_activity_feed` in the same file was not changed — it intentionally shows historical activity without a today-only filter.

---

# Issue #3 — The same song keeps showing up twice in search

## How you reproduced it

1. Read Simone's report: searching `Anthem` returned Crown Heights Anthem three times; songs with fewer tags appeared once.
2. Traced `GET /songs/search?q=Anthem` in `routes/songs.py` to `search_songs()` in `services/search_service.py`.
3. Ran the existing search tests before changing code:
   ```bash
   python -m pytest tests/test_search.py -v
   ```
   `test_search_no_duplicates_multi_tag_song` encodes Simone's scenario — a song with three tags searched by title.
4. Confirmed the SQL-level duplication with the buggy join. Seeded Crown Heights Anthem with tags `rap`, `hip-hop`, and `boom bap`, then ran the equivalent raw SQL:
   ```sql
   SELECT song.id FROM song
   LEFT OUTER JOIN song_tags ON song.id = song_tags.song_id
   WHERE song.title LIKE '%Anthem%'
   ```
   Result: **3 rows** for one song (one row per tag). A song with one tag returns 1 row; zero tags returns 1 row — matching Simone's "some once, others two or three times" pattern.

## How you found the root cause

1. Started at the route (`routes/songs.py` → `search`) and followed into `services/search_service.py`.
2. Noticed `search_songs` queries `Song` but also does `.outerjoin(song_tags, Song.id == song_tags.c.song_id)`.
3. Checked `Song.to_dict()` in `models.py` — tags are already loaded via the ORM relationship (`self.tags`), not from the join. The join serves no purpose for the response.
4. Understood join behavior: a `LEFT OUTER JOIN` on a many-to-many association table produces **one SQL row per matching tag**. Three tags → three joined rows for the same song. That is the multiplication Simone saw in search results.

## The root cause

`search_songs` included an unnecessary `.outerjoin(song_tags, ...)` on the `song_tags` association table. Because `song_tags` has one row per tag assignment, a song with N tags generates N joined SQL rows. The query then materialized one result per joined row, so Crown Heights Anthem (3 tags) appeared three times, while songs with 0–1 tags appeared once. Tags did not need the join — `song.to_dict()` already reads them from `self.tags`.

## Your fix and side-effect check

**Change:** Removed the `.outerjoin(song_tags, ...)` line and unused `Tag` / `song_tags` imports. The query now filters `Song` by title or artist only:

```python
results = (
    db.session.query(Song)
    .filter(
        db.or_(
            Song.title.ilike(f"%{query}%"),
            Song.artist.ilike(f"%{query}%"),
        )
    )
    .all()
)
```

**Why this fixes it:** Without the join, SQL returns one row per matching song regardless of tag count. Tags are still included in each result via `song.to_dict()` → `self.tags`.

**Side-effect check:** Ran search, feed, and streak tests after the fix:
```bash
python -m pytest tests/test_search.py tests/test_feed.py tests/test_streaks.py -v
```
All 12 tests pass. Verified manually that Crown Heights Anthem still returns all three tag names in its `tags` array after the fix. `get_song()` in the same file was unchanged.
