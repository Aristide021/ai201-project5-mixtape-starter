# Project 5: Mixtape Bug Hunt — Submission

## AI Usage

I used Claude Code (claude-sonnet-4-6) throughout this project. During codebase orientation, I had it read all service files and explain each module's responsibility and how they connect. For bug investigation, I read the relevant service files myself first to form hypotheses, then used Claude to confirm my understanding — for example, asking it to explain what `today.weekday()` returns vs. `isoweekday()` to verify the Issue #1 diagnosis, and to compare the `add_to_playlist` and `rate_song` code paths side-by-side to confirm Issue #4's missing notification. The fixes themselves came from direct code reading rather than AI suggestion — Claude helped verify reasoning but the root cause in each case was visible in the code once I knew where to look.

---

## Codebase Map

### Main Files and Their Roles

**`app.py`** — Flask app factory. Creates the Flask app, initializes SQLAlchemy, and registers all route blueprints. Running `create_app()` is the entry point.

**`models.py`** — Defines 6 SQLAlchemy models: `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus 3 association tables: `friendships` (User ↔ User many-to-many), `song_tags` (Song ↔ Tag many-to-many), and `playlist_entries` (Playlist ↔ Song many-to-many with a `position` integer for ordering and `added_by` FK).

**`routes/songs.py`** — Endpoints for sharing songs, searching, rating (`POST /songs/<id>/rate`), and recording listening events.

**`routes/playlists.py`** — Endpoints for creating playlists and retrieving songs in a playlist (`GET /playlists/<id>/songs`).

**`routes/users.py`** — Endpoints for user profiles, streaks, and notifications.

**`routes/feed.py`** — Endpoints for "Friends Listening Now" and the general activity feed.

**`services/streak_service.py`** — Manages listening streak increments and resets. Called by the listening event route.

**`services/feed_service.py`** — Filters listening events by recency and friendship to produce the "Listening Now" feed.

**`services/search_service.py`** — Queries songs matching a string against title and artist fields, joined with tags.

**`services/notification_service.py`** — Creates Notification records and handles rating/playlist actions, each of which should trigger a notification to the song's original sharer.

**`services/playlist_service.py`** — Creates playlists and queries their songs ordered by the `position` column in `playlist_entries`.

### Data Flow — User Rates a Song

1. Client sends `POST /songs/<song_id>/rate` with `{ "user_id": ..., "score": ... }`
2. `routes/songs.py` parses the request and calls `notification_service.rate_song(user_id, song_id, score)`
3. `rate_song()` validates the score, looks up the Song and User, upserts a Rating record, and (after the fix) calls `create_notification()` for the song's original sharer
4. `create_notification()` persists a `Notification` row with `notification_type="song_rated"` and a human-readable body
5. The sharer can retrieve it via `GET /users/<id>/notifications`

### Patterns

Every route delegates immediately to a service function. Routes handle request parsing and response formatting; all business logic lives in `services/`. This makes the bugs easy to isolate — if an endpoint misbehaves, the cause is almost always in the corresponding service file, not the route.

---

## Root Cause Analyses

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:** Set a user's `last_listened_at` to the prior Saturday (one day before Sunday), then called the listening event endpoint to simulate a listen on Sunday. The streak reset to 1 instead of incrementing.

**How I found the root cause:** I went directly to `streak_service.py` and read `update_listening_streak()`. The logic branch for `days_since_last == 1` had an extra guard: `and today.weekday() != 6`. I looked up what `weekday()` returns — 0=Monday through 6=Sunday — and confirmed the guard was explicitly preventing the streak from incrementing on Sundays.

**The root cause:** `datetime.weekday()` returns 6 for Sunday. The condition `today.weekday() != 6` evaluated to `False` every Sunday, so any Sunday listen that was exactly one day after the previous listen fell through to the `else` branch and reset the streak to 1. Every consecutive streak that crossed a Saturday→Sunday boundary was destroyed.

**Fix and side-effect check:** Removed the `and today.weekday() != 6` guard, leaving `elif days_since_last == 1:` alone. The other branches (`days_since_last == 0` for same-day no-ops and the `else` reset) are unchanged. I verified that a streak spanning Monday→Sunday→Monday now increments correctly on both Sunday and Monday.

---

### Issue #2 — Friends Listening Now shows people from yesterday

**How I reproduced it:** The seed data includes listening events from multiple days. Calling `GET /feed/listening-now` returned friends whose last listen was 20+ hours ago — clearly "yesterday" in any practical sense of "listening now."

**How I found the root cause:** `feed_service.py` defines `RECENT_THRESHOLD = timedelta(hours=24)` at the top of the file. The cutoff is `datetime.now(UTC) - RECENT_THRESHOLD`. With a 24-hour window, anyone who listened in the past day qualifies as "listening now," which includes listeners from yesterday evening being shown through the following afternoon.

**The root cause:** The threshold was set to 24 hours, which is the right window for a general activity feed but far too wide for a "listening now" feature. A user who listened 23 hours ago is not "listening now."

**Fix and side-effect check:** Changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)`. The `get_activity_feed()` function does not use this constant (it has no time filter by design), so only `get_friends_listening_now()` is affected.

---

### Issue #3 — The same song keeps showing up twice in search

**How I reproduced it:** Searched for a song that I knew had multiple tags (verifiable by checking the seed data). The same song appeared in results once per tag it had.

**How I found the root cause:** `search_service.py` joins `Song` with `song_tags` using `outerjoin`. A song with N tags produces N rows in the join result. Without `.distinct()`, the query returns one result row per tag, not per song. SQLAlchemy maps each row to a `Song` object, so the same song object appears multiple times in the list.

**The root cause:** The `outerjoin` on `song_tags` was needed to enable filtering by tag, but the query had no deduplication. Any song with 2 tags appeared twice; a song with 3 tags appeared three times. The current search only filters on title/artist (not on tag content), so the join is purely additive duplicates.

**Fix and side-effect check:** Added `.distinct()` to the query chain between the join and the filter. This collapses duplicate rows from multi-tag songs back to one result per song. The `song.to_dict()` call still loads tags via the `Song.tags` relationship (which uses `lazy="subquery"` and is unaffected by the query-level distinct).

---

### Issue #4 — Notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:** Used the API to have user A rate a song originally shared by user B, then retrieved user B's notifications. No `song_rated` notification was present, even though the rating was saved successfully.

**How I found the root cause:** I compared `add_to_playlist()` and `rate_song()` in `notification_service.py` side by side. `add_to_playlist()` ends with a `create_notification()` call guarded by `if song.shared_by != added_by_user_id`. `rate_song()` has no equivalent call — it saves the Rating and returns, with no notification step.

**The root cause:** The `rate_song()` function was never wired up to send a notification. The infrastructure (`create_notification()`, the `Notification` model, the notification retrieval endpoint) all existed and worked; the call was simply missing from `rate_song()`.

**Fix and side-effect check:** Added a `create_notification()` call at the end of `rate_song()`, mirroring the pattern in `add_to_playlist()`: notify `song.shared_by` with type `"song_rated"` and a message including the rater's username, song title, and score, skipped if the rater is the original sharer. The rating save and return are unchanged.

---

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:** Created a playlist with 3 songs. `GET /playlists/<id>/songs` returned only 2 songs — always missing the last one regardless of which song was added last.

**How I found the root cause:** `playlist_service.py:get_playlist_songs()` queries songs ordered by position, then returns `[song.to_dict() for song in songs[:-1]]`. The `[:-1]` Python slice drops the last element of any list.

**The root cause:** `songs[:-1]` is a Python list slice that returns all elements except the last one. This was presumably a typo or a copy-paste error from some other slicing context. Every playlist was missing its final song regardless of how many songs it contained.

**Fix and side-effect check:** Changed `songs[:-1]` to `songs` (no slice). The query, ordering, and `to_dict()` call are all unchanged. Playlists with 0 or 1 songs are unaffected (slicing an empty list or single-element list returned `[]` and `[]` before — now they correctly return `[]` and the one song).
