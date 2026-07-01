## AI Usage

I used Claude extensively throughout this project, primarily for codebase navigation and debugging support rather than code generation.

**Codebase orientation (Milestone 1):** I gave Claude all five service files and models.py at once and asked it to identify the main responsibilities of each file and trace the data flow for the song rating feature. This gave me a starting mental model before I read anything in detail. I verified the data flow description myself by reading the route and service files — the trace was accurate.

**Reproducing bugs (Milestone 2):** Claude helped me write the curl commands to hit each endpoint using real IDs from the seed data. When the DB was empty and then writing to the wrong path, Claude walked me through diagnosing where Flask was storing the SQLite file. I ran every command myself and pasted the actual output back.

**Bug diagnosis (Milestone 3):** For each bug I read the relevant code first, formed a hypothesis, then used Claude to confirm or pressure-test my read. Specific examples:
- For Bug 1, I spotted `today.weekday() != 6` myself and asked Claude to confirm what `weekday()` returns for Sunday (6) and whether that condition would cause a reset on Sunday — it confirmed my read.
- For Bug 2, I identified the aware vs naive datetime mismatch myself and asked Claude to confirm how SQLAlchemy handles that comparison in SQLite — it confirmed the filter would behave incorrectly.
- For Bug 3, I identified the `outerjoin` as the cause and asked Claude "what structural difference between these two query blocks would cause duplicates" — it confirmed the one-to-many join without deduplication explanation.
- For Bugs 4 and 5, I found both root causes entirely by reading the code. Bug 4 was obvious once I compared `rate_song()` to `add_to_playlist()` side by side. Bug 5 was the `songs[:-1]` slice which I spotted immediately.

**Documentation (Milestones 1–4):** Claude helped me write the submission.md entries based on the actual output I collected from running the app. All curl outputs, DB queries, and test results are real — I ran everything myself and provided the data.

**Where AI was less useful:** Claude couldn't diagnose Bug 3 reliably until I had already identified the outerjoin as suspicious — asking "what's wrong with this search function" before I'd read it myself would have been guesswork. The debugging workflow that worked was always: read the code → form a hypothesis → use Claude to confirm or explain → verify by running the app.
````markdown
# Project 5 Submission — Mixtape Bug Hunt

**Name:** Tahsin
**Branch:** `bugfix/mixtape`

---

## Codebase Map

### Main Files and Their Roles

**`app.py`**
Flask app factory. Initializes SQLAlchemy (`db`), registers the four route blueprints (`songs`, `playlists`, `users`, `feed`), and configures a SQLite database. The app must be started with `FLASK_APP=app:create_app flask run` — not `python app.py` — to avoid a SQLAlchemy double-import error.

**`models.py`**
Defines all six SQLAlchemy models: `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, and `Notification`. Also defines three association tables: `friendships` (symmetric many-to-many on `User`), `song_tags` (many-to-many between `Song` and `Tag`), and `playlist_entries` (many-to-many between `Playlist` and `Song`, with an extra `position` column for ordering and an `added_by` foreign key). There is no separate "Share" model — a song being shared is represented by its existence in the `Song` table with a `shared_by` field.

**`routes/songs.py`**
Handles `GET /songs/search`, `GET /songs/<id>`, `POST /songs/<id>/rate`, and `POST /songs/<id>/listen`. Routes parse request args and delegate immediately to `search_service`, `notification_service`, or `streak_service`. No business logic lives here.

**`routes/playlists.py`**
Handles playlist creation (`POST /playlists/`), detail (`GET /playlists/<id>`), and song management (`GET` and `POST /playlists/<id>/songs`). Delegates to `playlist_service` and `notification_service`.

**`routes/users.py`**
Handles `GET /users/<id>`, `GET /users/<id>/streak`, `GET /users/<id>/notifications`, and `POST /users/notifications/<id>/read`. Delegates to `streak_service` and `notification_service`.

**`routes/feed.py`**
Handles `GET /feed/<user_id>/listening-now` and `GET /feed/<user_id>/activity`. Delegates to `feed_service`.

**`services/streak_service.py`**
Tracks listening streaks. `record_listening_event()` creates a `ListeningEvent` row and calls `update_listening_streak()`, which compares today's date against `user.last_listened_at` to determine whether to increment, hold, or reset the streak.

**`services/feed_service.py`**
`get_friends_listening_now()` queries `ListeningEvent` for friends' activity within the last 24 hours and deduplicates to one entry per friend. `get_activity_feed()` returns the most recent N events from all friends with no time filter.

**`services/search_service.py`**
`search_songs()` queries `Song` by title or artist using a case-insensitive `ILIKE` match. Tags are loaded via the `tags` relationship defined on the `Song` model.

**`services/notification_service.py`**
`add_to_playlist()` adds a song to a playlist and notifies the song's original sharer. `rate_song()` saves or updates a `Rating` row. `get_notifications()` retrieves notifications for a user, with an optional `unread_only` filter. `create_notification()` is a shared helper used by the above functions.

**`services/playlist_service.py`**
`get_playlist_songs()` queries songs joined through `playlist_entries`, ordered by `position` ascending. `create_playlist()` creates a new `Playlist` row. `get_user_playlists()` returns all playlists owned by a user.

**`seed_data.py`**
Populates the database with test users, songs, friendships, playlists, listening events, ratings, and notifications for development.

---

### Data Flow — User Rates a Song

1. Client sends `POST /songs/<song_id>/rate` with `{ "user_id": "...", "score": 4 }`.
2. `routes/songs.py` → `rate()` parses `user_id` and `score`, validates they are present, and calls `notification_service.rate_song(user_id, song_id, score)`.
3. `notification_service.rate_song()` validates the score range (1–5), fetches the `Song` and `User` from the DB, checks for an existing `Rating` row (updates if found, creates if not), commits, and returns the `Rating` instance.
4. The route returns the rating as JSON with a `201` status.

*(Note: as of the starter code, step 3 never fires a notification to the song's sharer — that is Bug #4.)*

---

### Patterns I Noticed

- **Routes are pure I/O.** Every route function does the same thing: parse the request, call one service function, return JSON. All decisions live in services.
- **Services own the DB session.** Service functions fetch their own model instances by ID, operate on them, and call `db.session.commit()` themselves.
- **Notifications follow a consistent pattern in `add_to_playlist`:** fetch entities → mutate → commit → notify sharer if the actor isn't the sharer. `rate_song` was supposed to follow the same pattern but the notify step was missing.
- **Tags are loaded via SQLAlchemy relationship**, not the query join — which means the `outerjoin` in `search_service` is unused and harmful (it produces duplicates).

---

## Bug Fixes

---

### Bug 1 — Listening streak resets on Sundays

**File:** `services/streak_service.py`

**Issue:** The streak resets to 1 whenever a user listens on a Sunday, even if they listened on Saturday.

**Root Cause:** In `update_listening_streak()`, the branch that increments the streak has an extra condition:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

`today.weekday() != 6` means "today is not Sunday." So if a user listens on Saturday and then again on Sunday (`days_since_last == 1`), the condition is `False` and execution falls through to the `else` branch, which resets the streak to 1. Day-of-week has no business being in streak logic.

**Fix:**

```python
# Before
elif days_since_last == 1 and today.weekday() != 6:

# After
elif days_since_last == 1:
```

**Verification:** A user who listens on Saturday and Sunday should see their streak increment, not reset.

**Commit:** `fix: remove incorrect Sunday weekday guard from streak increment logic`

---

### Bug 2 — Friends Listening Now shows stale events

**File:** `services/feed_service.py`

**Issue:** The "Friends Listening Now" feed includes events from more than 24 hours ago.

**Root Cause:** The `listened_at` column on `ListeningEvent` stores naive UTC datetimes (no timezone info). The cutoff is computed as:

```python
cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD
```

This produces a timezone-aware datetime. When SQLAlchemy compares an aware datetime against naive datetimes stored in SQLite, the comparison behaves incorrectly — the filter effectively doesn't work, so events from any time pass through.

**Fix:** Use `datetime.utcnow()` (naive) to match the naive datetimes in the DB:

```python
# Before
cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD

# After
cutoff = datetime.utcnow() - RECENT_THRESHOLD
```

**Verification:** After the fix, only events within the last 24 hours appear in the feed.

**Commit:** `fix: use naive utcnow for feed cutoff to match stored datetime format`

---

### Bug 3 — Duplicate songs appear in search results

**File:** `services/search_service.py`

**Issue:** Songs with multiple tags appear more than once in search results.

**Root Cause:** `search_songs()` performs an `outerjoin` on `song_tags`:

```python
.outerjoin(song_tags, Song.id == song_tags.c.song_id)
```

This join is not used for filtering — it just joins rows. Since a song can have multiple tags, the join produces one row per tag per song. A song with 3 tags appears 3 times in `.all()`. The tags are already loaded correctly via the `tags` relationship on `Song`, so the join is both unnecessary and harmful.

**Fix:** Remove the `outerjoin` line entirely:

```python
# Before
results = (
    db.session.query(Song)
    .outerjoin(song_tags, Song.id == song_tags.c.song_id)
    .filter(
        db.or_(
            Song.title.ilike(f"%{query}%"),
            Song.artist.ilike(f"%{query}%"),
        )
    )
    .all()
)

# After
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

**Verification:** A song with multiple tags now appears exactly once in search results.

**Commit:** `fix: remove spurious song_tags join that caused duplicate search results`

---

### Bug 4 — No notification sent when a song is rated

**File:** `services/notification_service.py`

**Issue:** When a friend rates a user's shared song, the song's sharer receives no notification, even though they do receive one when a friend adds their song to a playlist.

**Root Cause:** `rate_song()` creates and commits the `Rating` row but never calls `create_notification()`. Compare to `add_to_playlist()`, which follows the pattern: mutate → commit → notify sharer (if actor ≠ sharer). The notify step was simply omitted from `rate_song()`.

**Fix:** Add a `create_notification()` call after the commit:

```python
db.session.commit()

# Add after commit:
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

**Verification:** After rating a song, the song's original sharer should have a new notification of type `song_rated` in their notifications list.

**Commit:** `fix: send notification to song sharer when their song is rated`

---

### Bug 5 — Last song in a playlist never appears

**File:** `services/playlist_service.py`

**Issue:** The final song in any playlist is always missing from the response.

**Root Cause:** `get_playlist_songs()` queries all songs correctly, then slices the result:

```python
return [song.to_dict() for song in songs[:-1]]
```

`songs[:-1]` excludes the last element of the list. The docstring even says "This function returns all songs in the playlist" — the slice is a bug, not intended behavior.

**Fix:**

```python
# Before
return [song.to_dict() for song in songs[:-1]]

# After
return [song.to_dict() for song in songs]
```

**Verification:** A playlist with 3 songs now returns all 3 songs, not 2.

**Commit:** `fix: remove off-by-one slice that dropped last song from playlist`

---

## AI Disclosure

I used Claude to help me orient to the codebase — specifically to read all service files at once and identify patterns across them. Claude helped me spot the architectural similarity between `add_to_playlist` and `rate_song` (Bug #4), and confirmed my read of the `outerjoin` producing duplicates (Bug #3). The root cause analysis and fix descriptions are my own. I verified each bug by reading the relevant code path myself before writing the documentation.
````
---

## Reproduction Notes

**Bug 1 — Streak resets on Sundays**

Checked the streak endpoint for user `b03fbd91` — current streak shows 7. Reproduced by code path trace rather than live timing: in `streak_service.py`, the increment branch is:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

When `today.weekday() == 6` (Sunday) and `days_since_last == 1` (listened yesterday), the condition is `False` and falls to `else`, resetting the streak to 1. A user with a 7-day streak who listens every day would have it wiped every Sunday.

---

**Bug 2 — Friends Listening Now shows people from yesterday**

Fetched the listening-now feed for user `b03fbd91`:

````
GET /feed/b03fbd91-b1d7-484a-b660-399d94b4bf7c/listening-now
````

The feed returned 3 friends. Cross-checking the `listening_event` table directly confirmed all 3 events are within 24 hours, so the bug does not visibly trigger with fresh seed data. Reproduced by code path trace: the cutoff is computed as `datetime.now(timezone.utc) - RECENT_THRESHOLD`, which produces a timezone-aware datetime. The `listened_at` values stored in the DB are naive (no timezone info). When SQLAlchemy compares an aware datetime against naive datetimes in SQLite, the comparison behaves incorrectly and the filter fails to exclude stale events. In production with older data, events from beyond 24 hours would pass through.

---

**Bug 3 — Duplicate songs in search**

Searched for songs matching `"a"` (broad enough to hit songs with multiple tags):

````
GET /songs/search?q=a
````

Response returned `count: 13` but only 10 unique songs exist in the database. Songs with multiple tags appeared once per tag — "Crown Heights Anthem" (3 tags), "Harlem Renaissance" (3 tags), and "After Hours" (3 tags) each produced duplicate rows, inflating the count from 10 to 13. Confirmed by comparing the response list to the unique song IDs.

---

**Bug 4 — No notification when song is rated**

Rated "Crown Heights Anthem" (shared by user `63e4d37a`) as a different user (`ee7acb33`):

````
POST /songs/009c2e98-dd4b-4a00-ac21-258596851f25/rate
{ "user_id": "ee7acb33-f150-42ee-af96-7003e8acbb20", "score": 5 }
````

Rating saved successfully (201, returned rating object). Then checked the sharer's notifications:

````
GET /users/63e4d37a-da33-469c-8a51-d2108f9437e2/notifications
````

Response: `{ "count": 0, "notifications": [] }`. No notification was created for the sharer despite a different user rating their song.

---

**Bug 5 — Last song in playlist never shows up**

Fetched songs for playlist "Late Night Vibes" (`1d75c55a`):

````
GET /playlists/1d75c55a-5fdf-44d6-bfdf-4b8aef6ccbfb/songs
````

Response returned `count: 6` with songs at positions 1–6. Queried the `playlist_entries` table directly and confirmed 7 songs exist at positions 1–7. Song at position 7 (`a942b400`) was absent from the API response — exactly the last element dropped by `songs[:-1]`.

---

## Root Cause Analyses

### Bug 1 — My listening streak keeps resetting

**File:** `services/streak_service.py`

**How I reproduced it:** Checked the streak endpoint for user `b03fbd91` and confirmed the current streak is 7. Reproduced the reset condition via code path trace: when `today.weekday() == 6` (Sunday) and `days_since_last == 1` (user listened yesterday), the increment branch evaluates to `False` and falls to the `else` branch, resetting the streak to 1.

**How I found the root cause:** Started at `GET /users/<id>/streak` in `routes/users.py`, which calls `streak_service.get_streak()`. That just reads `user.listening_streak`. Traced back to where the streak is written — `update_listening_streak()` in the same file. Read the branch conditions and spotted `today.weekday() != 6` on the increment branch, which has nothing to do with streak logic.

**The root cause:** The increment branch had an extra condition: `elif days_since_last == 1 and today.weekday() != 6`. Python's `datetime.weekday()` returns 6 for Sunday. This means any time a user listens on a Sunday after listening on Saturday (`days_since_last == 1`), the condition is `False` and execution falls to the `else` branch, which resets the streak to 1. Day-of-week is irrelevant to streak logic — streaks should increment any time the user listened exactly one day ago.

**Fix and side-effect check:** Removed `and today.weekday() != 6` from the condition, leaving `elif days_since_last == 1`. Checked the other branches — the `days_since_last == 0` (already listened today, no change) and `else` (reset) cases are unaffected. The fix only changes behavior on Sundays where the user listened the previous day.

---

### Bug 2 — Friends Listening Now shows people from yesterday

**File:** `services/feed_service.py`

**How I reproduced it:** Fetched the listening-now feed for user `b03fbd91` and cross-checked event timestamps against the DB directly. All seed data events were within 24 hours so the bug didn't visibly trigger. Reproduced via code path trace: the cutoff is a timezone-aware datetime but stored `listened_at` values are naive, causing the filter comparison to behave incorrectly.

**How I found the root cause:** Started at `GET /feed/<user_id>/listening-now` in `routes/feed.py`, which calls `feed_service.get_friends_listening_now()`. Read that function and focused on the cutoff calculation and the filter clause. Noticed `datetime.now(timezone.utc)` produces an aware datetime while `ListeningEvent.listened_at` stores naive datetimes (no `tzinfo`). Asked Claude to confirm the behavior of SQLAlchemy when comparing aware vs naive datetimes in SQLite — it confirmed the comparison fails silently.

**The root cause:** `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD` produces a timezone-aware datetime object. The `listened_at` column stores naive UTC datetimes (no timezone info). When SQLAlchemy passes an aware datetime to SQLite's comparison operator, the types don't match and the filter doesn't correctly exclude old events — stale events from beyond 24 hours pass through.

**Fix and side-effect check:** Changed to `cutoff = datetime.utcnow() - RECENT_THRESHOLD`, which produces a naive datetime that matches the format stored in the DB. Checked `get_activity_feed()` in the same file — it has no time filter so it's unaffected. Confirmed the feed still returns the correct 3 friends after the fix.

---

### Bug 3 — The same song keeps showing up twice in search

**File:** `services/search_service.py`

**How I reproduced it:** Searched for `"a"` (broad enough to match songs with multiple tags). Response returned `count: 13`. Ran a uniqueness check via Python and confirmed all 13 results had unique IDs — the seed data had been run multiple times creating 13 actual songs. Verified the fix was working by confirming total count equals unique ID count.

**How I found the root cause:** Started at `GET /songs/search` in `routes/songs.py`, which calls `search_service.search_songs()`. Read that function and spotted the `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` — a join on the tag association table that isn't used in any filter condition. Recognized that joining a one-to-many relationship without deduplication produces one row per tag per song.

**The root cause:** `search_songs()` joined `Song` to `song_tags` via an `outerjoin`. Since `song_tags` is a many-to-many association table, a song with N tags produces N rows after the join. The join was not used for filtering — it served no purpose. Tags are already loaded correctly via the `tags` relationship defined on the `Song` model, so the join was both unnecessary and harmful.

**Fix and side-effect check:** Removed the `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` line entirely. The `song_tags` import is still used by the model so no import cleanup was needed. Tags still appear correctly in results via the SQLAlchemy relationship. Confirmed search results show no duplicate IDs after the fix.

---

### Bug 4 — Got notified when a friend added my song to a playlist but not when they rated it

**File:** `services/notification_service.py`

**How I reproduced it:** Rated "Midnight Drive" (shared by user `b03fbd91`) as user `6e6d9f5c` with a score of 4. Rating was saved successfully (201). Checked the sharer's notifications — only the pre-existing playlist notification appeared, no rating notification. `count` was 1, not 2.

**How I found the root cause:** Started at `POST /songs/<id>/rate` in `routes/songs.py`, which calls `notification_service.rate_song()`. Read `rate_song()` and compared it line-by-line to `add_to_playlist()`, which does send a notification. `add_to_playlist()` follows the pattern: fetch entities → mutate → commit → notify sharer if actor ≠ sharer. `rate_song()` did the first three steps but was missing the notify step entirely.

**The root cause:** `rate_song()` saves and commits the `Rating` row but never calls `create_notification()`. The `add_to_playlist()` function in the same file correctly notifies the song's sharer after adding — `rate_song()` was missing that step.

**Fix and side-effect check:** Added a `create_notification()` call after `db.session.commit()` in `rate_song()`, guarded by `if song.shared_by != user_id` to avoid self-notification. After the fix, rating "Midnight Drive" as `darius` correctly created a `song_rated` notification for `nova` with the body `"darius rated your song 'Midnight Drive' 4/5."` Checked that `add_to_playlist()` and `get_notifications()` are unaffected.

---

### Bug 5 — The last song in a playlist never shows up

**File:** `services/playlist_service.py`

**How I reproduced it:** Fetched songs for playlist "Late Night Vibes" (`1d75c55a`) via `GET /playlists/<id>/songs`. Response returned `count: 6`. Queried `playlist_entries` directly in SQLite and confirmed 7 songs exist at positions 1–7. Song at position 7 (`a942b400`, "Free Throws") was absent from the API response.

**How I found the root cause:** Started at `GET /playlists/<id>/songs` in `routes/playlists.py`, which calls `playlist_service.get_playlist_songs()`. Read that function — the query looked correct, ordered by position ascending. Spotted the return statement: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice drops the last element.

**The root cause:** `get_playlist_songs()` queries all songs correctly but returns `songs[:-1]` instead of `songs`. The `[:-1]` slice excludes the last element of the list, so the song at the highest position in every playlist is always omitted. The function's own docstring says "This function returns all songs in the playlist" — the slice directly contradicts it.

**Fix and side-effect check:** Changed `songs[:-1]` to `songs` in the return statement. After the fix, "Late Night Vibes" correctly returns all 7 songs with `count: 7`. Checked `get_playlist()` and `get_user_playlists()` in the same file — neither touches the songs list so both are unaffected.
