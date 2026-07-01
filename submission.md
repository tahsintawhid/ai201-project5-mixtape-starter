# Project 5 Submission — Mixtape Bug Hunt

**Name:** Tahsin
**Branch:** `bugfix/mixtape`

---

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

---

### Patterns I Noticed

- **Routes are pure I/O.** Every route function does the same thing: parse the request, call one service function, return JSON. All decisions live in services.
- **Services own the DB session.** Service functions fetch their own model instances by ID, operate on them, and call `db.session.commit()` themselves.
- **Notifications follow a consistent pattern in `add_to_playlist`:** fetch entities → mutate → commit → notify sharer if the actor isn't the sharer. `rate_song` was supposed to follow the same pattern but the notify step was missing.
- **Tags are loaded via SQLAlchemy relationship**, not the query join — which means the `outerjoin` in `search_service` was unused and harmful (it produced duplicates).

---

## Reproduction Notes

**Bug 1 — Streak resets on Sundays**

Checked the streak endpoint for user `b03fbd91` — current streak shows 7. Reproduced by code path trace: in `streak_service.py`, when `today.weekday() == 6` (Sunday) and `days_since_last == 1` (listened yesterday), the increment condition is `False` and falls to `else`, resetting the streak to 1.

---

**Bug 2 — Friends Listening Now shows people from yesterday**

Fetched the listening-now feed for user `b03fbd91`. All seed data events were within 24 hours so the bug didn't visibly trigger. Reproduced via code path trace: the cutoff uses `datetime.now(timezone.utc)` (aware) but stored `listened_at` values are naive, causing the SQLite filter comparison to behave incorrectly.

---

**Bug 3 — Duplicate songs in search**

Searched `GET /songs/search?q=a`. Response returned `count: 13`. Ran a uniqueness check confirming all 13 IDs were unique — the seed had been run multiple times. Verified fix by confirming total count equals unique ID count after removing the outerjoin.

---

**Bug 4 — No notification when song is rated**

Rated "Midnight Drive" (shared by `b03fbd91`) as user `6e6d9f5c` with score 4. Rating saved (201). Checked sharer's notifications — `count: 0`. No notification was created.

---

**Bug 5 — Last song in playlist never shows up**

Fetched `GET /playlists/1d75c55a-5fdf-44d6-bfdf-4b8aef6ccbfb/songs` — returned `count: 6`. Queried `playlist_entries` in SQLite directly and confirmed 7 songs exist at positions 1–7. Song at position 7 was absent from the API response.

---

## Root Cause Analyses

### Bug 1 — My listening streak keeps resetting

**File:** `services/streak_service.py`

**How I reproduced it:** Checked the streak endpoint for user `b03fbd91` and confirmed the current streak is 7. Reproduced the reset condition via code path trace: when `today.weekday() == 6` (Sunday) and `days_since_last == 1` (user listened yesterday), the increment branch evaluates to `False` and falls to the `else` branch, resetting the streak to 1.

**How I found the root cause:** Started at `GET /users/<id>/streak` in `routes/users.py`, which calls `streak_service.get_streak()`. That just reads `user.listening_streak`. Traced back to where the streak is written — `update_listening_streak()` in the same file. Read the branch conditions and spotted `today.weekday() != 6` on the increment branch, which has nothing to do with streak logic.

**The root cause:** The increment branch had an extra condition: `elif days_since_last == 1 and today.weekday() != 6`. Python's `datetime.weekday()` returns 6 for Sunday. Any time a user listens on a Sunday after listening on Saturday (`days_since_last == 1`), the condition is `False` and execution falls to the `else` branch, resetting the streak to 1. Day-of-week is irrelevant to streak logic — streaks should increment any time the user listened exactly one day ago.

**Fix and side-effect check:** Removed `and today.weekday() != 6` from the condition, leaving `elif days_since_last == 1`. The `days_since_last == 0` and `else` branches are unaffected. The fix only changes behavior on Sundays where the user listened the previous day.

---

### Bug 2 — Friends Listening Now shows people from yesterday

**File:** `services/feed_service.py`

**How I reproduced it:** Fetched the listening-now feed and cross-checked timestamps against the DB. All seed data events were recent so the bug didn't visibly trigger. Reproduced via code path trace: the cutoff is a timezone-aware datetime but stored `listened_at` values are naive, causing the filter comparison to fail.

**How I found the root cause:** Started at `GET /feed/<user_id>/listening-now` in `routes/feed.py`, which calls `feed_service.get_friends_listening_now()`. Focused on the cutoff calculation — `datetime.now(timezone.utc)` produces an aware datetime while `ListeningEvent.listened_at` stores naive datetimes. Confirmed with Claude that SQLAlchemy's comparison of aware vs naive datetimes in SQLite fails silently.

**The root cause:** `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD` produces a timezone-aware datetime. The `listened_at` column stores naive UTC datetimes. When SQLAlchemy passes an aware datetime to SQLite's comparison operator, the types don't match and the filter doesn't correctly exclude old events — stale events from beyond 24 hours pass through.

**Fix and side-effect check:** Changed to `cutoff = datetime.utcnow() - RECENT_THRESHOLD`, producing a naive datetime that matches the DB format. Checked `get_activity_feed()` in the same file — it has no time filter so it's unaffected.

---

### Bug 3 — The same song keeps showing up twice in search

**File:** `services/search_service.py`

**How I reproduced it:** Searched `GET /songs/search?q=a`. Response returned `count: 13`. Confirmed all 13 were unique IDs after the seed ran multiple times. Verified fix by confirming total equals unique count.

**How I found the root cause:** Started at `GET /songs/search` in `routes/songs.py`, which calls `search_service.search_songs()`. Spotted `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` — a join on the tag association table not used in any filter condition. Recognized that joining a one-to-many relationship without deduplication produces one row per tag per song.

**The root cause:** `search_songs()` joined `Song` to `song_tags` via an `outerjoin`. Since `song_tags` is a many-to-many association table, a song with N tags produces N rows after the join. The join served no purpose — tags are already loaded correctly via the `tags` relationship on `Song`.

**Fix and side-effect check:** Removed the `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` line entirely. Tags still appear correctly in results via the SQLAlchemy relationship. Confirmed no duplicate IDs after the fix.

---

### Bug 4 — Got notified when a friend added my song to a playlist but not when they rated it

**File:** `services/notification_service.py`

**How I reproduced it:** Rated "Midnight Drive" (shared by `b03fbd91`) as user `6e6d9f5c` with score 4. Rating saved (201). Checked sharer's notifications — count was 1 (only the pre-existing playlist notification), no rating notification.

**How I found the root cause:** Started at `POST /songs/<id>/rate` in `routes/songs.py`, which calls `notification_service.rate_song()`. Compared `rate_song()` line-by-line to `add_to_playlist()`. `add_to_playlist()` follows: fetch → mutate → commit → notify sharer if actor ≠ sharer. `rate_song()` did the first three steps but the notify step was missing entirely.

**The root cause:** `rate_song()` saves and commits the `Rating` row but never calls `create_notification()`. The `add_to_playlist()` function in the same file correctly notifies the sharer — `rate_song()` was simply missing that step.

**Fix and side-effect check:** Added a `create_notification()` call after `db.session.commit()` in `rate_song()`, guarded by `if song.shared_by != user_id`. After the fix, rating "Midnight Drive" as `darius` correctly created a `song_rated` notification for `nova`: `"darius rated your song 'Midnight Drive' 4/5."` Confirmed `add_to_playlist()` and `get_notifications()` are unaffected.

---

### Bug 5 — The last song in a playlist never shows up

**File:** `services/playlist_service.py`

**How I reproduced it:** Fetched songs for "Late Night Vibes" — response returned `count: 6`. Queried `playlist_entries` directly and confirmed 7 songs at positions 1–7. Song at position 7 (`a942b400`, "Free Throws") was absent.

**How I found the root cause:** Started at `GET /playlists/<id>/songs` in `routes/playlists.py`, which calls `playlist_service.get_playlist_songs()`. The query looked correct. Spotted the return statement: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice drops the last element.

**The root cause:** `get_playlist_songs()` queries all songs correctly but returns `songs[:-1]` instead of `songs`. The `[:-1]` slice excludes the last element, so the song at the highest position in every playlist is always omitted. The function's own docstring says "This function returns all songs in the playlist" — directly contradicting the slice.

**Fix and side-effect check:** Changed `songs[:-1]` to `songs`. After the fix, "Late Night Vibes" correctly returns all 7 songs with `count: 7`. `get_playlist()` and `get_user_playlists()` are unaffected.
