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
