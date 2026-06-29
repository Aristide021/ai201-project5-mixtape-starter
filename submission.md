# Project 5: Mixtape Bug Hunt — Submission

## AI Usage

**Instance 1 — Codebase orientation.** I pasted each service file into Claude and asked "What is this module responsible for? What are its main functions and what does each one do?" I used this to build an initial mental model before reading the code myself. Claude's summaries were accurate at the module level but missed the interaction between functions — for example, it described `rate_song()` as handling ratings without flagging that it doesn't send a notification, because it was only looking at one function in isolation. I caught that later by reading both functions side by side.

**Instance 2 — Confirming a date API detail (Issue #1).** Once I had narrowed the streak bug to the `today.weekday() != 6` condition, I asked Claude to confirm what `datetime.weekday()` returns for each day of the week. It confirmed 0=Monday through 6=Sunday. I verified this in a Python REPL before making the fix. This saved me from having to look it up in documentation, but the diagnosis — that the Sunday exclusion was wrong — was mine before I asked.

**Instance 3 — Structural comparison (Issue #4).** After reading both `add_to_playlist()` and `rate_song()`, I asked Claude to describe the structural difference between the two functions. It identified the missing `create_notification()` call correctly. I had already spotted the same thing myself; I used Claude to sanity-check that I wasn't missing some other path where the notification might be sent elsewhere in the codebase.

**Instance 4 — Threshold value (Issue #2).** After deciding the 24-hour threshold was wrong, I asked Claude what a reasonable window for "listening now" would be. It suggested 15 minutes. I used 30 minutes instead after re-reading the feature description, which says "recently" not "right now." I did not accept Claude's suggestion directly.

---

## Codebase Map

### Main Files and Their Roles

**`app.py`** — Flask app factory. Initializes SQLAlchemy and registers the four route blueprints (songs, playlists, users, feed). The entry point is `create_app()` — the README warns not to run `python app.py` directly because it triggers a double-import error with SQLAlchemy.

**`models.py`** — Defines 7 SQLAlchemy models and 3 association tables:
- `User` — has `listening_streak` (int) and `last_listened_at` (datetime) stored directly on the user row, not derived
- `Song` — has `shared_by` FK pointing to the User who originally shared it; this is what notification logic uses to find who to notify
- `Tag` — simple name table, linked to Song via `song_tags` (many-to-many)
- `ListeningEvent` — one row per listen, with `user_id`, `song_id`, `listened_at`
- `Rating` — one row per (user, song) pair with a `UNIQUE` constraint; score is 1–5
- `Playlist` — has `is_collaborative` boolean; linked to Song via `playlist_entries`
- `Notification` — has `notification_type` (string), `body` (text), `read` (bool)
- `playlist_entries` — the join table between Playlist and Song; adds `position` (int) and `added_by` (FK to User), so songs have an explicit order and we know who added each one

**`routes/songs.py`** — `POST /songs` (share), `GET /songs/search`, `POST /songs/<id>/listen`, `POST /songs/<id>/rate`

**`routes/playlists.py`** — `POST /playlists` (create), `POST /playlists/<id>/songs` (add song), `GET /playlists/<id>/songs` (get ordered songs)

**`routes/users.py`** — `GET /users/<id>` (profile + streak), `GET /users/<id>/notifications`, `PATCH /notifications/<id>/read`

**`routes/feed.py`** — `GET /feed/listening-now` (friends active recently), `GET /feed/activity` (general recent feed)

**`services/streak_service.py`** — `record_listening_event()` creates a ListeningEvent and calls `update_listening_streak()`. The streak logic lives entirely in `update_listening_streak()`: it compares today's date to `user.last_listened_at` and decides whether to increment, leave unchanged, or reset.

**`services/feed_service.py`** — `get_friends_listening_now()` filters ListeningEvents to those within a `RECENT_THRESHOLD` window from friends. `get_activity_feed()` does the same without a time filter.

**`services/search_service.py`** — `search_songs()` does a case-insensitive `ilike` match on `title` and `artist`, with an `outerjoin` to `song_tags`.

**`services/notification_service.py`** — `add_to_playlist()` adds a song to a playlist and calls `create_notification()` for the sharer. `rate_song()` upserts a Rating record. `get_notifications()` retrieves a user's notifications ordered by recency.

**`services/playlist_service.py`** — `get_playlist_songs()` queries songs joined via `playlist_entries`, ordered by `position` ascending.

### Data Flow — User Rates a Song

1. Client sends `POST /songs/<song_id>/rate` with body `{ "user_id": "...", "score": 4 }`
2. `routes/songs.py` extracts `user_id` and `score` from the JSON body and calls `notification_service.rate_song(user_id, song_id, score)`
3. `rate_song()` validates score is 1–5, fetches the Song and User from the DB, then checks for an existing Rating row with `filter_by(user_id=user_id, song_id=song_id)`
4. If a Rating exists it updates `score`; otherwise it creates a new Rating and adds it to the session
5. `db.session.commit()` persists the rating
6. The route returns the rating dict to the client

Note: there is no notification sent in this flow — the notification for ratings is missing (this is Issue #4).

### Patterns I Noticed

Every route delegates immediately to a service function. Routes only do input parsing (`request.get_json()`) and response formatting (`jsonify`). All database access and business logic lives in `services/`. This means if an endpoint returns wrong data, the bug is almost certainly in the service, not the route.

The notification pattern in `add_to_playlist()` is: do the main action, commit, then call `create_notification()` for the song's sharer if they weren't the one who triggered the action. This same pattern is absent from `rate_song()`.

---

## Root Cause Analyses

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:**

```python
# flask shell
from models import User
from services.streak_service import update_listening_streak
from datetime import datetime, timezone, timedelta

user = User.query.first()
# Simulate: user last listened on Saturday
user.last_listened_at = datetime(2026, 6, 27, 12, 0, tzinfo=timezone.utc)  # Saturday
user.listening_streak = 5

# Simulate: user listens on Sunday (next day)
sunday = datetime(2026, 6, 28, 14, 0, tzinfo=timezone.utc)
update_listening_streak(user, sunday)
print(user.listening_streak)  # prints 1 — should be 6
```

The streak reset to 1 even though the user listened on consecutive days (Saturday then Sunday).

**How I found the root cause:**

I started at the route: `POST /songs/<id>/listen` in `routes/songs.py` calls `streak_service.record_listening_event()`. I read `record_listening_event()` — it creates a ListeningEvent then calls `update_listening_streak(user, now)`. I read `update_listening_streak()` top to bottom. The three branches are: `days_since_last == 0` (no-op), `days_since_last == 1` (increment), else (reset). I almost stopped there thinking it looked correct, but then I noticed the second branch had an extra condition: `days_since_last == 1 and today.weekday() != 6`. I didn't immediately know what `weekday()` returns for Sunday, so I checked — it returns 6. That confirmed: the increment branch was being skipped every Sunday, and any listen on Sunday with `days_since_last == 1` fell through to the reset.

**The root cause:**

`datetime.weekday()` returns 6 for Sunday. The condition `today.weekday() != 6` evaluates to `False` on every Sunday. So when a user listened Saturday and then again Sunday (`days_since_last == 1`), the full condition `days_since_last == 1 and today.weekday() != 6` was `True and False = False`. Execution fell to the `else` branch and reset the streak to 1. Any streak that crossed a Saturday→Sunday boundary was destroyed.

**Fix and side-effect check:**

Removed `and today.weekday() != 6` from the condition, leaving just `elif days_since_last == 1:`. The `days_since_last == 0` (same-day) and `else` (gap > 1 day) branches are unchanged. After the fix, running the flask shell reproduction above prints 6 as expected. I also verified the `days_since_last == 0` path still returns early (no change) and the reset path still fires when `days_since_last > 1`.

---

### Issue #2 — Friends Listening Now shows people from yesterday

**How I reproduced it:**

The seed data includes ListeningEvents with `listened_at` timestamps from multiple days ago. After seeding:

```bash
curl http://127.0.0.1:5000/feed/listening-now?user_id=<user_id>
```

The response included friends whose `listened_at` was 18–23 hours ago. These users weren't listening "now" by any reasonable definition.

**How I found the root cause:**

I went to `routes/feed.py` first and found it calls `feed_service.get_friends_listening_now(user_id)`. I opened `feed_service.py`. The first thing I saw was `RECENT_THRESHOLD = timedelta(hours=24)` at the top of the file. The cutoff is `datetime.now(UTC) - RECENT_THRESHOLD`. I read the query: it filters `ListeningEvent.listened_at >= cutoff`. With a 24-hour cutoff, any listen from the past day qualifies. I compared `get_friends_listening_now()` to `get_activity_feed()` below it — `get_activity_feed()` has no time filter by design ("returns the most recent N events regardless of when they happened"). The threshold constant is only used by the "listening now" function, but it's set to a value that's appropriate for an activity feed, not a real-time presence indicator.

**The root cause:**

`RECENT_THRESHOLD` was set to `timedelta(hours=24)`. This made "Friends Listening Now" include anyone who listened in the past 24 hours — which includes people who listened yesterday evening. The feature name implies recent/current activity (minutes, not hours). A user who listened 23 hours ago is not "listening now."

**Fix and side-effect check:**

Changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)`. I checked that `get_activity_feed()` does not reference `RECENT_THRESHOLD` — it does not, so the general activity feed is unaffected. After the change, the same `curl` command returns an empty list for seed data older than 30 minutes, and returns a friend if I first POST a fresh listening event for them.

---

### Issue #3 — The same song keeps showing up twice in search

**How I reproduced it:**

```bash
curl "http://127.0.0.1:5000/songs/search?q=neon"
```

The response contained the same song object (same `id`, same `title`) appearing multiple times. Cross-referencing with the seed data, the duplicated songs were ones that had multiple tags. A song with 2 tags appeared twice; one with 3 tags appeared three times.

**How I found the root cause:**

I went to `routes/songs.py`, found the search endpoint calls `search_service.search_songs(query)`, and opened `search_service.py`. The query starts with `db.session.query(Song)` then does `.outerjoin(song_tags, Song.id == song_tags.c.song_id)`. I paused on that join — `outerjoin` on a many-to-many table means each tag a song has produces a separate row in the result set. With no `.distinct()` call, SQLAlchemy returns one `Song` object per row, not per unique song. I confirmed by counting: the number of duplicate appearances matched the number of tags on that song.

I also checked whether removing the join would break anything. The filter only checks `Song.title.ilike()` and `Song.artist.ilike()` — it doesn't filter on tags at all. The join serves no filtering purpose; it was likely added anticipating tag-based search that was never implemented.

**The root cause:**

The `outerjoin(song_tags)` produces N rows for a song with N tags. Without `.distinct()`, the query returns N copies of the same Song ORM object. `search_songs()` then calls `song.to_dict()` on each, producing N identical dicts in the response list.

**Fix and side-effect check:**

Added `.distinct()` to the query between the join and the filter. This collapses duplicate rows back to one result per song. The `song.to_dict()` call loads tags via `Song.tags` (a separate `lazy="subquery"` relationship), which is not affected by `.distinct()` on the main query — tags still appear correctly on each result. After the fix, the same search returns each song exactly once regardless of how many tags it has.

---

### Issue #4 — Notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:**

```bash
# User A rates a song originally shared by User B
curl -X POST http://127.0.0.1:5000/songs/<song_id>/rate \
  -H "Content-Type: application/json" \
  -d '{"user_id": "<user_a_id>", "score": 5}'

# Check User B's notifications
curl http://127.0.0.1:5000/users/<user_b_id>/notifications
```

The rating was saved (the POST returned a valid rating dict), but User B's notifications list was empty. Repeating with `add_to_playlist` produced a notification correctly.

**How I found the root cause:**

I opened `notification_service.py` and read both `add_to_playlist()` and `rate_song()` side by side. `add_to_playlist()` ends with:

```python
if song.shared_by != added_by_user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_added_to_playlist",
        body=f"...",
    )
```

`rate_song()` ends with `db.session.commit()` and `return rating` — no notification call. The infrastructure (`create_notification()`, the Notification model, the retrieval endpoint) all existed and worked. The call was simply never added to `rate_song()`.

**The root cause:**

`rate_song()` in `notification_service.py` saves the rating but never calls `create_notification()`. The `add_to_playlist()` function in the same file follows the correct pattern — notify the sharer unless they were the one who triggered the action. `rate_song()` is missing that entire block.

**Fix and side-effect check:**

Added a `create_notification()` call at the end of `rate_song()`, after `db.session.commit()` and before `return rating`, mirroring the guard from `add_to_playlist()`:

```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

The rating upsert logic and return value are unchanged. I verified: after the fix, User B receives a `song_rated` notification when User A rates their song, and no notification is created when a user rates their own song.

---

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:**

```bash
# Get songs in a playlist that has 3 songs in seed data
curl http://127.0.0.1:5000/playlists/<playlist_id>/songs
```

The response contained only 2 songs. I confirmed via flask shell that the playlist had 3 entries in `playlist_entries`. The missing song was always the one with the highest `position` value — always the last one.

**How I found the root cause:**

I went to `routes/playlists.py`, found `GET /playlists/<id>/songs` calls `playlist_service.get_playlist_songs(playlist_id)`, and read that function in `playlist_service.py`. The query looked correct — it joins through `playlist_entries`, filters by `playlist_id`, orders by `position` ascending. Then I hit the return line:

```python
return [song.to_dict() for song in songs[:-1]]
```

`songs[:-1]` is a Python slice that returns all elements of a list *except the last one*. The query was returning the right songs in the right order; the slice was discarding the final one before returning.

**The root cause:**

`songs[:-1]` silently drops the last element of the list on every call. A playlist with 3 songs returns 2; a playlist with 1 song returns 0. This affected every playlist regardless of size or content.

**Fix and side-effect check:**

Changed `songs[:-1]` to `songs`. The query, join, filter, and ordering are all unchanged. After the fix, all three songs are returned for a 3-song playlist. Edge cases: an empty playlist still returns `[]`; a 1-song playlist now correctly returns that one song (previously returned nothing).
