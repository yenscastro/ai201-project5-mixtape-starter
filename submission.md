# Project 5 — Mixtape Bug Hunt: Submission

**Branch:** `bugfix/mixtape`
**Bugs fixed:** 3 of 5 (Issues #1, #4, #5)

All three fixes live in the `services/` layer. Each was reproduced before fixing and verified afterward against the seeded database.

---

## Issue #1 — Listening streak keeps resetting

**Affected service:** `services/streak_service.py`

**Symptom:** A user's listening streak fails to increment when their consecutive listening day falls on a Sunday. Listening Saturday then Sunday leaves the streak flat instead of going up by one.

**Root cause:** The consecutive-day branch carried an extra Sunday guard, `and today.weekday() != 6`, so the increment was skipped whenever "today" was a Sunday (`weekday() == 6`). There is no business reason to exclude Sundays — a consecutive calendar day is a consecutive day regardless of which day of the week it is.

**Fix** (`update_listening_streak`, line 73):

```python
# before
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1

# after
elif days_since_last == 1:
    user.listening_streak += 1
```

**Verification:** Set kenji's `last_listened_at` to Sat 2026-06-13 with `listening_streak = 12`, then called `update_listening_streak` with Sun 2026-06-14.
- Confirmed 2026-06-14 is a Sunday.
- Result: streak `12 → 13` (before the fix it stayed at 12).

---

## Issue #5 — Last song in a playlist never shows up

**Affected service:** `services/playlist_service.py`

**Symptom:** A playlist with N songs only ever returns N−1 through `GET /playlists/<id>/songs`; the final song (highest position) is always missing.

**Root cause:** `get_playlist_songs` sliced the ordered result with `songs[:-1]`, dropping the last element of an already-correct, position-ordered query.

**Fix** (`get_playlist_songs`, line 66):

```python
# before
return [song.to_dict() for song in songs[:-1]]

# after
return [song.to_dict() for song in songs]
```

**Verification:** `GET /playlists/82f0060c-.../songs` (Late Night Vibes, seeded with 7 songs).
- Before: `"count": 6` (missing "Free Throws", position 7).
- After: `"count": 7`, all songs present including "Free Throws".

---

## Issue #4 — Notified when a song is added to a playlist, but not when it's rated

**Affected service:** `services/notification_service.py`

**Symptom:** When a friend adds your shared song to a playlist you get a notification, but when a friend *rates* your song nothing happens. `rate_song` saved the rating and returned without ever notifying the song's original sharer.

**Root cause:** `rate_song` had no notification logic at all — unlike `add_to_playlist`, which already notified `song.shared_by`. The feature was simply never wired up.

**Fix** (`rate_song`, after the rating commits):

```python
db.session.commit()

# Notify the person who originally shared the song (if it wasn't them who rated it)
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score} stars.",
    )

return rating
```

This mirrors the existing pattern in `add_to_playlist`, including the guard that suppresses a notification when someone rates their own song.

**Verification:** kenji rated simone's song "Crown Heights Anthem" (`10933452-...`) with score 5.
- Simone's notifications: `0 → 1`.
- Notification body: `kenji rated your song 'Crown Heights Anthem' 5 stars.` (type `song_rated`).
- Confirmed via `GET /users/<simone>/notifications` → `"count": 1`.
- Self-rating guard checked: simone rating her own song creates no notification.

---

## Not fixed (out of scope for this submission)

- **Issue #2** — Friends Listening Now shows people from yesterday (`feed_service.py`)
- **Issue #3** — The same song shows up twice in search (`search_service.py`)

## How to reproduce the verification

```bash
python seed_data.py                              # reseed (note: regenerates all UUIDs)
FLASK_APP=app:create_app flask run               # start the server
pytest tests/                                     # run the test suite
```
