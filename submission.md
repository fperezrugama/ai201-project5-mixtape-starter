# AI Usage / Collaboration Statement

This project was completed with the help of an AI coding assistant (Claude Code). I used it as a collaborator throughout while directing the overall process, reviewing its output, and making the final decisions. This is an honest account of how AI was and was not used — it did substantial analysis, coding, and drafting under my direction; I did not write every line myself, and I did not let it act unreviewed.

**How I used AI**
- **Understanding the codebase (Milestone 1):** I had it explore the repository, summarize each file, and explain the architecture, models, services, and routes. The codebase map in this document was produced with its help.
- **Explaining unfamiliar code and tracing execution:** I asked it to walk through call chains (route → service → model) and explain how specific features work.
- **Reproducing the bugs (Milestone 2):** I used it to run the test suite and write throwaway scripts against the seeded database, and to explain what the failing tests and outputs meant. It also helped me compare all five reported issues and decide which three were the most reliably reproducible.
- **Locating root causes (Milestone 3):** It helped trace each bug to a specific line — for example, sweeping every weekday transition for the streak bug and doing a layer-by-layer trace for the playlist bug — and explained why each defect produced the observed behavior.
- **Discussing and implementing fixes:** I discussed candidate fixes with it and had it apply the smallest change for each bug.
- **Documentation:** It drafted the reproduction notes and root-cause write-ups in this file, which I reviewed for accuracy.
- **Git history cleanup:** It helped me split a commit that accidentally contained two fixes into one commit per bug.

![Git log showing separate fix commits](Screenshot 2026-07-05 at 12.52.13 AM.png)

**How I verified the work rather than just trusting the AI**
- I ran the full `pytest` suite and confirmed all 13 tests pass before accepting each fix.
- I checked that each fix was scoped to a single bug and left unrelated behavior unchanged.
- Before the rewritten Git history was force-pushed, I confirmed the resulting code was byte-for-byte identical to the original.

**Where I corrected or questioned the AI**
- I required an investigation-only pass that proved each root cause with evidence *before* any code was changed, and had a premature fix reverted so the process was followed in order.
- A commit that had been labeled with the wrong message was identified and corrected.
- I decided how the Git history should be restructured and explicitly approved the force-push rather than letting it happen automatically.

---

# Milestone 1 — Codebase Orientation

_Mixtape: a Flask + SQLAlchemy social music-sharing app. This document is a mental model of the codebase only — no bugs are investigated or fixed here._

---

## 1. Application Architecture

### Overall architecture
Mixtape is a **monolithic Flask REST API** built on the classic **three-tier layered architecture**:

```
HTTP client
    │
    ▼
Routes layer (Flask Blueprints)      ← HTTP concerns: parse request, validate presence, serialize JSON, map errors → status codes
    │
    ▼
Services layer (plain functions)     ← business logic: streaks, feeds, search, notifications, playlists
    │
    ▼
Models layer (SQLAlchemy ORM)        ← data schema, relationships, to_dict() serialization
    │
    ▼
Database (SQLite: mixtape.db)
```

The app uses the **application-factory pattern** (`create_app()` in `app.py`) with a module-level `db = SQLAlchemy()` instance that is bound to the app via `db.init_app(app)`. Blueprints are registered inside the factory with URL prefixes.

### Layer responsibilities
| Layer | Responsibility | Does NOT do |
|-------|----------------|-------------|
| **Routes** | Read `request` args/JSON, check required fields, call one service function, `jsonify` the result, translate `ValueError` → 400/404 | Business logic, direct DB writes (mostly) |
| **Services** | All domain logic and DB queries; raise `ValueError` for "not found"/validation; commit transactions | Touch `request`/`response` objects |
| **Models** | Define tables, columns, relationships, association tables, and `to_dict()` serializers | Business logic |

### Dependency flow
Dependencies point **downward only**:
- Routes import from Services (and occasionally `models`/`db` directly).
- Services import from `models` and `app.db`.
- Models import `db` from `app`.

No upward imports (services never import routes; models never import services). The one lateral dependency is `notification_service` importing `playlist_service` (and `Playlist`) lazily inside a function to avoid a circular import at module load.

---

## 2. Folder Structure

| Path | Purpose |
|------|---------|
| **`/` (root)** | App entry points and configuration: `app.py`, `models.py`, `seed_data.py`, `requirements.txt`, `README.md`. |
| **`routes/`** | HTTP layer. One Flask Blueprint module per resource (`songs`, `playlists`, `users`, `feed`). Thin controllers that delegate to services. |
| **`services/`** | Business-logic layer. One module per domain concern (`streak`, `feed`, `search`, `notification`, `playlist`). This is where the project brief says the bugs live. |
| **`tests/`** | `pytest` unit tests targeting the service layer directly (streaks, search, playlists). Use an in-memory SQLite DB via fixtures. |

Both `routes/` and `services/` contain an empty `__init__.py` marking them as importable packages.

---

## 3. Major Files

| File | Responsibility | Key contents |
|------|----------------|--------------|
| **`app.py`** | Flask application factory + DB setup | `db = SQLAlchemy()`; `create_app(config=None)` — configures DB URI (`DATABASE_URL` env or `sqlite:///mixtape.db`), registers the 4 blueprints with prefixes, calls `db.create_all()`. |
| **`models.py`** | All ORM models + association tables | `generate_uuid()` helper; association tables `friendships`, `song_tags`, `playlist_entries`; models `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`; each model has a `to_dict()`. |
| **`seed_data.py`** | Populate DB with realistic test data | `seed()` — drops/recreates all tables; inserts 5 users, friendships, 10 tags, 13 songs (0/1/3-tag groups), listening events (recent + old), 3 playlists, and a sample notification. |
| **`routes/songs.py`** | Song search, detail, rating, listening endpoints | `songs_bp`; endpoints `search`, `get_song_detail`, `rate`, `listen`. |
| **`routes/playlists.py`** | Playlist create/detail/songs endpoints | `playlists_bp`; endpoints `create`, `get_detail`, `get_songs`, `add_song`. |
| **`routes/users.py`** | User profile, streak, notification endpoints | `users_bp`; endpoints `get_user`, `streak`, `notifications`, `read_notification`. |
| **`routes/feed.py`** | Social feed endpoints | `feed_bp`; endpoints `listening_now`, `activity`. |
| **`services/streak_service.py`** | Listening-streak logic | `record_listening_event`, `update_listening_streak`, `get_streak`. |
| **`services/feed_service.py`** | Friends-listening-now + activity feed | `get_friends_listening_now`, `get_activity_feed`; module constant `RECENT_THRESHOLD = 24h`. |
| **`services/search_service.py`** | Song search | `search_songs`, `get_song`. |
| **`services/notification_service.py`** | Notifications + rating + add-to-playlist | `create_notification`, `add_to_playlist`, `rate_song`, `get_notifications`, `mark_as_read`. |
| **`services/playlist_service.py`** | Playlist create/retrieval | `create_playlist`, `get_playlist_songs`, `get_playlist`, `get_user_playlists`. |
| **`tests/test_streaks.py`** | Streak unit tests | 5 tests covering new user, consecutive day, same-day, skipped day, weekend boundary. |
| **`tests/test_search.py`** | Search unit tests | 5 tests covering matching, duplicate-avoidance across tag counts, empty results. |
| **`tests/test_playlists.py`** | Playlist unit tests | 3 tests covering all-songs, ordering, empty playlist. |
| **`requirements.txt`** | Dependencies | `flask`, `flask-sqlalchemy`, `sqlalchemy`, `python-dotenv`, `pytest`. |

---

## 4. Database Models

All primary keys are 36-char UUID strings (`generate_uuid()`). All timestamps default to `datetime.now(timezone.utc)`.

### Association tables (not full models)
| Table | Columns | Purpose |
|-------|---------|---------|
| **`friendships`** | `user_id` (FK→user), `friend_id` (FK→user) | Many-to-many **self-referential** friendship. Symmetric by convention — seed inserts both directions. |
| **`song_tags`** | `song_id` (FK→song), `tag_id` (FK→tag) | Many-to-many between songs and tags. |
| **`playlist_entries`** | `playlist_id` (FK→playlist), `song_id` (FK→song), `position` (int, not null), `added_by` (FK→user), `added_at` (datetime) | Many-to-many between playlists and songs **with extra data** (ordering + provenance). This is why it isn't a bare join table. |

### Models
| Model | Key fields | Foreign keys | Relationships |
|-------|-----------|--------------|---------------|
| **`User`** | `username` (unique), `email` (unique), `listening_streak` (int, default 0), `last_listened_at` (nullable), `created_at` | — | `shared_songs`→Song (backref `shared_by_user`); `ratings`→Rating; `listening_events`→ListeningEvent; `notifications`→Notification; `playlists`→Playlist; `friends`→User (self-M2M via `friendships`, `lazy="dynamic"`). |
| **`Tag`** | `name` (unique) | — | Referenced by Song via `song_tags`. |
| **`Song`** | `title`, `artist`, `album` (nullable), `genre` (nullable), `shared_at`, `share_note` (nullable) | `shared_by`→user | `ratings`→Rating; `listening_events`→ListeningEvent; `tags`→Tag (M2M via `song_tags`, `lazy="subquery"`). |
| **`ListeningEvent`** | `listened_at` | `user_id`→user, `song_id`→song | backrefs `listener` (User), `song` (Song). Records a single play; basis for streaks and feeds. |
| **`Rating`** | `score` (int 1–5), `rated_at` | `user_id`→user, `song_id`→song | backref `rater` (User), `song` (Song). **`UniqueConstraint(user_id, song_id)`** — one rating per user per song. |
| **`Playlist`** | `name`, `created_at`, `is_collaborative` (bool, default True) | `created_by`→user | `creator` backref (User); `songs`→Song (M2M via `playlist_entries`, `lazy="subquery"`). |
| **`Notification`** | `notification_type` (str), `body` (text), `created_at`, `read` (bool, default False) | `user_id`→user (recipient) | backref `recipient` (User). |

### Entity-relationship summary
- A **User** shares many Songs, creates many Playlists, has many Ratings / ListeningEvents / Notifications, and has many friends (other Users).
- A **Song** is shared by one User, can be rated/listened-to many times, carries many Tags, and can appear in many Playlists.
- A **Playlist** is created by one User and contains many ordered Songs (through `playlist_entries`).

---

## 5. Services Layer

| Service module | Public functions | Models / tables used | Called by (routes) |
|----------------|------------------|----------------------|--------------------|
| **`streak_service`** | `record_listening_event(user_id, song_id)` → ListeningEvent; `update_listening_streak(user, now)` → None (helper, mutates user); `get_streak(user_id)` → int | `User`, `ListeningEvent` | `songs.listen` (record); `users.streak` (get_streak) |
| **`feed_service`** | `get_friends_listening_now(user_id)` → list[dict]; `get_activity_feed(user_id, limit=20)` → list[dict] | `User`, `Song`, `ListeningEvent` | `feed.listening_now`; `feed.activity` |
| **`search_service`** | `search_songs(query)` → list[dict]; `get_song(song_id)` → dict | `Song`, `Tag`, `song_tags` | `songs.search`; `songs.get_song_detail` |
| **`notification_service`** | `create_notification(user_id, type, body)` → Notification; `add_to_playlist(playlist_id, song_id, added_by)` → None; `rate_song(user_id, song_id, score)` → Rating; `get_notifications(user_id, unread_only=False)` → list[dict]; `mark_as_read(notification_id)` → None | `Notification`, `Song`, `User`, `Rating`, `Playlist` (lazy import) | `playlists.add_song` (add_to_playlist); `songs.rate` (rate_song); `users.notifications` (get_notifications); `users.read_notification` (mark_as_read) |
| **`playlist_service`** | `create_playlist(name, created_by, is_collaborative=True)` → Playlist; `get_playlist_songs(playlist_id)` → list[dict]; `get_playlist(playlist_id)` → dict; `get_user_playlists(user_id)` → list[dict] | `Playlist`, `Song`, `User`, `playlist_entries` | `playlists.create`; `playlists.get_songs`; `playlists.get_detail`. (`get_user_playlists` is imported in `routes/playlists.py` but not currently wired to an endpoint.) |

**Cross-service dependency:** `notification_service.add_to_playlist` imports `playlist_service.get_playlist_songs` and the `Playlist` model lazily (function-local) to avoid circular imports.

**Convention:** every service that takes an entity ID looks it up with `db.session.get(...)` and raises `ValueError(f"{Entity} {id} not found")` if missing; routes catch that and return 404/400.

---

## 6. Routes Layer

Four blueprints, registered in `app.py` with URL prefixes.

### `songs_bp` — prefix `/songs`
| Method & path | Endpoint fn | Delegates to |
|---------------|-------------|--------------|
| `GET /songs/search?q=` | `search` | `search_service.search_songs` |
| `GET /songs/<song_id>` | `get_song_detail` | `search_service.get_song` |
| `POST /songs/<song_id>/rate` | `rate` | `notification_service.rate_song` |
| `POST /songs/<song_id>/listen` | `listen` | `streak_service.record_listening_event` |

### `playlists_bp` — prefix `/playlists`
| Method & path | Endpoint fn | Delegates to |
|---------------|-------------|--------------|
| `POST /playlists/` | `create` | `playlist_service.create_playlist` |
| `GET /playlists/<playlist_id>` | `get_detail` | `playlist_service.get_playlist` |
| `GET /playlists/<playlist_id>/songs` | `get_songs` | `playlist_service.get_playlist_songs` |
| `POST /playlists/<playlist_id>/songs` | `add_song` | `notification_service.add_to_playlist` |

### `users_bp` — prefix `/users`
| Method & path | Endpoint fn | Delegates to |
|---------------|-------------|--------------|
| `GET /users/<user_id>` | `get_user` | _direct_ `db.session.get(User, ...)` (no service) |
| `GET /users/<user_id>/streak` | `streak` | `streak_service.get_streak` |
| `GET /users/<user_id>/notifications?unread_only=` | `notifications` | `notification_service.get_notifications` |
| `POST /users/notifications/<notification_id>/read` | `read_notification` | `notification_service.mark_as_read` |

### `feed_bp` — prefix `/feed`
| Method & path | Endpoint fn | Delegates to |
|---------------|-------------|--------------|
| `GET /feed/<user_id>/listening-now` | `listening_now` | `feed_service.get_friends_listening_now` |
| `GET /feed/<user_id>/activity` | `activity` | `feed_service.get_activity_feed` |

---

## 7. Data Flow Example — Rating a Song

Tracing `POST /songs/<song_id>/rate` end to end:

```
HTTP Request
    │  POST /songs/<song_id>/rate
    │  Body: { "user_id": "...", "score": 5 }
    ▼
Route  (routes/songs.py → rate)
    │  • data = request.get_json()
    │  • extract user_id, score
    │  • validate both present → else 400
    │  • call rate_song(user_id, song_id, int(score))
    ▼
Service  (notification_service.rate_song)
    │  • validate 1 ≤ score ≤ 5 → else raise ValueError
    │  • db.session.get(Song, song_id)  → 404 if missing
    │  • db.session.get(User, user_id)  → 404 if missing
    │  • query Rating for (user_id, song_id)
    │       - if exists → update score
    │       - else      → create new Rating, session.add
    │  • db.session.commit()
    │  • return Rating instance
    ▼
Models / Database
    │  • Rating row upserted in SQLite
    │  • UniqueConstraint(user_id, song_id) enforces one rating per pair
    ▼
Response
       • route: rating.to_dict()  → jsonify → HTTP 201
       • error path: ValueError → { "error": ... } → HTTP 400
```

**Data transformations:** JSON body → Python dict (route) → validated primitives → ORM `Rating` object (service) → DB row (SQLAlchemy) → back to plain dict via `to_dict()` → JSON response.

---

## 8. Architectural Patterns

- **Separation of concerns:** HTTP handling (routes) is cleanly split from business logic (services) and persistence (models). Routes never write raw SQL; services never touch `request`/`response`.
- **Service-layer pattern:** business logic lives in stateless module-level functions grouped by domain, callable from routes and directly from tests.
- **Application-factory pattern:** `create_app()` builds and configures the app, enabling test configs (in-memory SQLite) via the `config` argument.
- **Blueprint-per-resource:** each URL namespace is an isolated blueprint registered with a prefix.
- **ORM serialization convention:** every model exposes `to_dict()`; services return lists/dicts of primitives, so routes only need `jsonify`.
- **Uniform error contract:** services raise `ValueError` for not-found/validation; routes translate to `{"error": ...}` with 400 (bad input) or 404 (missing resource).
- **Shared helpers / conventions:**
  - `generate_uuid()` as the default PK generator across all models.
  - `db.session.get(Model, id)` as the standard single-entity lookup.
  - UTC-aware timestamps via `datetime.now(timezone.utc)`.
  - Association tables used both as bare join tables (`song_tags`, `friendships`) and as an attributed join table (`playlist_entries`, which carries `position`, `added_by`, `added_at`).
  - Lazy in-function imports to break the `notification_service` ↔ `playlist_service` cycle.

---

## 9. Dependency Graph

```
                 ┌─────────────────────────────────────────┐
                 │                 Routes                   │
                 │  songs.py  playlists.py  users.py  feed.py│
                 └───────────────┬─────────────────────────┘
                                 │ (imports & calls)
                                 ▼
                 ┌─────────────────────────────────────────┐
                 │                Services                  │
                 │  streak  feed  search  notification  playlist
                 │        notification ──lazy──▶ playlist   │
                 └───────────────┬─────────────────────────┘
                                 │ (imports)
                                 ▼
                 ┌─────────────────────────────────────────┐
                 │            Models (models.py)            │
                 │  User Tag Song ListeningEvent Rating     │
                 │  Playlist Notification + assoc tables    │
                 └───────────────┬─────────────────────────┘
                                 │ (db = SQLAlchemy in app.py)
                                 ▼
                 ┌─────────────────────────────────────────┐
                 │        Database  (SQLite: mixtape.db)    │
                 └─────────────────────────────────────────┘
```

Flow is strictly top-to-bottom. `app.py` sits beside this stack: it owns `db` and wires blueprints together.

---

## 10. Investigation Index

| Feature | Route(s) | Service function(s) | Models involved |
|---------|----------|---------------------|-----------------|
| **Authentication** | _none_ — the app has no auth layer; caller identity is passed as `user_id` in the path/body | — | `User` (referenced, not authenticated) |
| **Songs** | `GET /songs/search`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen` | `search_service.search_songs` / `get_song`; `notification_service.rate_song`; `streak_service.record_listening_event` | `Song`, `Tag`, `song_tags`, `Rating`, `ListeningEvent`, `User` |
| **Playlists** | `POST /playlists/`, `GET /playlists/<id>`, `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs` | `playlist_service.create_playlist` / `get_playlist` / `get_playlist_songs`; `notification_service.add_to_playlist` | `Playlist`, `Song`, `playlist_entries`, `User`, `Notification` |
| **Feed** | `GET /feed/<id>/listening-now`, `GET /feed/<id>/activity` | `feed_service.get_friends_listening_now` / `get_activity_feed` | `User` (+ `friendships`), `ListeningEvent`, `Song` |
| **Notifications** | `GET /users/<id>/notifications`, `POST /users/notifications/<id>/read` | `notification_service.get_notifications` / `mark_as_read` / `create_notification` | `Notification`, `User` |
| **Search** | `GET /songs/search` | `search_service.search_songs` | `Song`, `Tag`, `song_tags` |
| **Streaks** | `POST /songs/<id>/listen`, `GET /users/<id>/streak` | `streak_service.record_listening_event` / `update_listening_streak` / `get_streak` | `User`, `ListeningEvent` |
| **Users** | `GET /users/<id>` | _direct model access_ | `User` |

> Note: There is no authentication/authorization feature — endpoints trust a `user_id` supplied by the caller. Worth keeping in mind, but out of scope for this milestone.

---

## 11. Codebase Map

```
ai201-project5-mixtape-starter/
│
├── app.py                 ── Flask factory create_app(); owns db = SQLAlchemy();
│                             registers 4 blueprints (/songs /playlists /users /feed); db.create_all()
│
├── models.py              ── 7 models + 3 association tables; UUID PKs; UTC timestamps; to_dict() serializers
│   ├── friendships (assoc)    User ↔ User  (symmetric)
│   ├── song_tags   (assoc)    Song ↔ Tag
│   ├── playlist_entries       Playlist ↔ Song  (+ position, added_by, added_at)
│   ├── User        streak fields, friends (self-M2M), owns songs/playlists/ratings/notifications
│   ├── Tag         name
│   ├── Song        title/artist/album/genre, shared_by → User, tags, ratings, plays
│   ├── ListeningEvent  user_id, song_id, listened_at  (streaks + feeds)
│   ├── Rating      user_id, song_id, score 1–5, UNIQUE(user_id, song_id)
│   ├── Playlist    created_by → User, is_collaborative, songs (M2M ordered)
│   └── Notification user_id, type, body, read
│
├── routes/               ── HTTP layer (thin controllers → services)
│   ├── songs.py      /songs      search · detail · rate · listen
│   ├── playlists.py  /playlists  create · detail · get songs · add song
│   ├── users.py      /users      profile · streak · notifications · mark read
│   └── feed.py       /feed       listening-now · activity
│
├── services/             ── business logic (bugs live here per the brief)
│   ├── streak_service.py       record_listening_event · update_listening_streak · get_streak
│   ├── feed_service.py         get_friends_listening_now · get_activity_feed   (RECENT_THRESHOLD=24h)
│   ├── search_service.py       search_songs · get_song
│   ├── notification_service.py create_notification · add_to_playlist · rate_song · get_notifications · mark_as_read
│   └── playlist_service.py     create_playlist · get_playlist_songs · get_playlist · get_user_playlists
│
├── tests/                ── pytest, in-memory SQLite fixtures, target services directly
│   ├── test_streaks.py    5 tests   ├── test_search.py  5 tests   └── test_playlists.py  3 tests
│
├── seed_data.py          ── seed(): drop+create; 5 users, friendships, 10 tags, 13 songs, 3 playlists, events, 1 notification
├── requirements.txt      ── flask, flask-sqlalchemy, sqlalchemy, python-dotenv, pytest
└── README.md             ── setup, run, test instructions + issue tracker

Request lifecycle:  HTTP → Blueprint route → service function → SQLAlchemy models → SQLite → to_dict() → JSON
Dependency flow:    Routes → Services → Models → Database   (strictly downward)
Error contract:     service raises ValueError → route returns {"error": ...} with 400/404
```

---

_Milestone 1 complete. No bugs were investigated, and no source code was modified._

---
---

# Milestone 2 — Reproduce the Bugs Before Fixing Them

_Objective: reliably reproduce bugs and document exactly how they occur. No source code was modified, no root causes were investigated, and no fixes were implemented. Reproduction evidence comes from running the existing `pytest` suite and from exercising the public service functions against the seeded database — observing outputs only._

## Issue Summary

| # | Issue | Feature | Conditions to trigger | Difficulty | Recommended? |
|---|-------|---------|-----------------------|------------|:---:|
| 1 | My listening streak keeps resetting | Streaks | Consecutive-day listen where the second day is a **Sunday** | Easy (deterministic via dated test) | **Yes** |
| 2 | Friends Listening Now shows people from yesterday | Feed | A friend whose **most-recent** listen is older than "now" but within 24h, with no newer event | Hard (seed data doesn't create this state) | No |
| 3 | The same song shows up twice in search | Search | A song matching the query that has **multiple tags** | Hard (does not reproduce in current stack) | No |
| 4 | Notified on playlist-add but not on rating | Notifications | A user rates a song shared by a **different** user | Easy (deterministic via seed data) | **Yes** |
| 5 | The last song in a playlist never shows | Playlists | Any playlist with ≥1 song | Easy (deterministic via test + seed) | **Yes** |

## Selected Bugs

**Chosen: Issues #1 (streak), #4 (notification on rate), #5 (playlist last song).**

These three were selected because each reproduces **deterministically and independently**, with reproduction confirmed by more than one method:

- **#5 — Last song in playlist** — The strongest candidate. Reproduces via **two failing unit tests** *and* against **all three seeded playlists** (each has 7 entries in the DB but only 6 are returned). No special conditions, no timing, always reproduces.
- **#1 — Streak resetting** — Reproduces deterministically through a **dated unit test** that pins a Saturday→Sunday transition, so the "conditional" (Sunday) nature is captured without relying on the real calendar. The neighboring consecutive-day test (Mon→Tue) passes, cleanly isolating the condition.
- **#4 — No notification on rating** — Reproduces deterministically against **seed data**: a friend rating another user's song produces **zero** notifications for the sharer, while the "song added to playlist" notification path is known to work (a seeded example exists). Contained to one feature and independent of the others.

**Why the other two were not selected:**

- **#3 (search duplicates)** — Could **not** be reproduced. The multi-tag song "Crown Heights Anthem" returns **exactly once** both through the `search_songs` service against seed data and through all five `test_search.py` tests (all pass, including the duplicate-specific ones). Reproduction would require conditions not present in this environment.
- **#2 (feed shows yesterday)** — Could **not** be reproduced with the provided seed data. Every one of nova's friends has a fresh listening event (~10–20 minutes old), and the feed keeps only each friend's most-recent listen, so it correctly shows recent listeners. The older "yesterday" events (10h/18h/26h ago) exist in the data but are superseded; triggering the report would require a friend whose *only* recent-ish listen falls in the "yesterday but < 24h" window, a state the seed script does not create.

---

## Reproduction Documentation

### Issue #5 — The last song in a playlist never shows up

**Preconditions**
- Application state: a seeded database (`python seed_data.py`), or the in-memory test fixtures.
- Required seed data: any playlist containing at least one song. All three seeded playlists ("Late Night Vibes", "Friday Energy", "Study Mode") each contain 7 songs.
- Required user: none specific.

**Steps to Reproduce**
1. Seed the database: `python seed_data.py`.
2. Run the playlist tests: `pytest tests/test_playlists.py -v`.
3. Observe `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` fail.
4. (Second, independent confirmation) Call `get_playlist_songs(playlist_id)` for any seeded playlist and count the results against the number of `playlist_entries` rows for that playlist.

**Expected Behavior**
- A playlist with 7 entries returns 7 songs; a fixture playlist with 5 songs returns all 5, in position order (`Track 1 … Track 5`).

**Actual Behavior**
- Each playlist returns **one fewer** song than it contains — the entry at the last position is missing:
  - `test_playlist_returns_all_songs`: expected 5, got 4.
  - `test_playlist_returns_songs_in_order`: expected `[Track 1..Track 5]`, got `[Track 1..Track 4]` (Track 5 dropped).
  - Seed confirmation — every playlist: `entries in DB=7, get_playlist_songs returned=6`.

**Reproducibility**
- **Always.** Reproduces on every playlist with ≥1 song, via both the test suite and seed data.

**Notes**
- The empty-playlist test (`test_empty_playlist_returns_empty_list`) passes.
- Observation only: the shortfall is consistently exactly one song, the one at the highest position. (No root-cause analysis performed.)

---

### Issue #1 — My listening streak keeps resetting

**Preconditions**
- Application state: test environment (in-memory SQLite fixture in `tests/test_streaks.py`).
- Required seed data: none — the test creates its own user.
- Required condition: two consecutive calendar days where the **second day is a Sunday** (`weekday() == 6`).

**Steps to Reproduce**
1. Run the streak tests: `pytest tests/test_streaks.py -v`.
2. Observe `test_streak_increments_on_sunday` fail. It records a listen on Saturday 2024-06-15, then Sunday 2024-06-16, and asserts the streak becomes 2.

**Expected Behavior**
- Listening on Saturday then the following Sunday should **increment** the streak: `1 → 2`.

**Actual Behavior**
- The streak does **not** increment on the Saturday→Sunday transition: it stays/resets to `1` (`assert 1 == 2` fails).

**Reproducibility**
- **Conditional.** Reproduces only when the second consecutive day is a Sunday. The equivalent non-Sunday case (`test_streak_increments_on_consecutive_day`, Monday→Tuesday) **passes**, which isolates the condition to Sunday specifically.

**Notes**
- Other streak tests pass: new-user starts at 1, same-day does not double-count, skipped-day resets to 1.
- Observation only: the failure is tied to the day-of-week of the second listen; no implementation was examined to determine why.

---

### Issue #4 — Notified when a friend adds my song to a playlist, but not when they rate it

**Preconditions**
- Application state: seeded database (`python seed_data.py`).
- Required seed data: at least two users and a song shared by one of them (e.g., "Crown Heights Anthem", shared by `simone`).
- Required user: a rater who is a **different** user from the song's sharer (e.g., `nova`).

**Steps to Reproduce**
1. Seed the database: `python seed_data.py`.
2. Note the sharer's current notification count via `get_notifications(sharer_id)` (for `simone` this is 0).
3. Have a different user rate the song: `rate_song(rater_id, song_id, 5)`.
4. Re-check the sharer's notification count.

**Expected Behavior**
- Rating another user's song notifies the sharer, mirroring the existing "song added to playlist" notification — the sharer's notification count should increase by 1.

**Actual Behavior**
- The sharer's notification count is **unchanged** (before=0, after=0) — **no notification is created** when the song is rated.

**Reproducibility**
- **Always** (given a rater different from the sharer). Deterministic against seed data.

**Notes**
- The complementary path is known to work: the seed script creates a `song_added_to_playlist` notification, and a working notification for that event exists for `nova`.
- Observation only: the rating itself succeeds (a `Rating` is returned); only the notification is absent. No root-cause analysis performed.

---

## Milestone 2 Checkpoint

- ✓ **Three bugs have been reproduced** — Issues #5, #1, and #4, each confirmed by at least two independent methods (test suite and/or seed-data observation).
- ✓ **No source code has been modified** — `git status` shows only the new `submission.md`; the database was restored to a clean seed afterward.
- ✓ **No debugging or root-cause investigation has been performed** — reproduction observed inputs/outputs only; no reasoning about *why* the bugs occur is included.
- ✓ **No fixes have been implemented.**

_Additional finding worth flagging (reproduction only, no diagnosis): Issues #2 and #3 could not be reproduced in this environment — #3 returns no duplicates through either the service or its tests, and #2's seed data always yields genuinely-recent feed entries. Recommend the three reproducible issues above._

_Milestone 2 complete. Awaiting Milestone 3 instructions before inspecting any implementation or attempting any fixes._

---
---

# Milestone 3A — Investigate, Fix, and Document Issue #5

_Scope: Issue #5 only (the last song in a playlist never shows up). No other issues were investigated, and no unrelated code was changed._

## 1. Investigation Log

1. **Started from the Milestone 2 reproduction.** Confirmed the reported behavior: every seeded playlist has 7 entries in the database but `get_playlist_songs` returns only 6, and two unit tests (`test_playlist_returns_all_songs`, `test_playlist_returns_songs_in_order`) fail. Condition: any playlist with ≥1 song. Reproducibility: always.
2. **Traced the execution path** from the affected route (`GET /playlists/<id>/songs`) down to the service. Only one service function is involved: `playlist_service.get_playlist_songs`.
3. **Read the implementation** of `get_playlist_songs`. The function (a) validates the playlist exists, (b) runs a SQL query joining `Song` to `playlist_entries`, filtered by playlist and ordered by `position`, then (c) returns `[song.to_dict() for song in songs[:-1]]`.
4. **Isolated the query from the slice.** Replicated the exact query inline against the seeded "Late Night Vibes" playlist *without* the `[:-1]` slice. The query returned **all 7 songs** in correct position order (`Midnight Drive … Free Throws`). Applying `[:-1]` dropped exactly the last one (`Free Throws`). This proved the query is correct and the slice is the sole cause.
5. **Applied the smallest fix** (removed the slice) and verified via the test suite and seed data, including boundary cases (empty playlist, single-song playlist).
6. **Removed all temporary debug scripts** (they were standalone throwaway scripts run against the DB; no debug code was ever added to source files).

## 2. Execution Trace

`GET /playlists/<playlist_id>/songs`

| Step | Function / location | Responsibility | Inputs | Outputs | Relevant? |
|------|---------------------|----------------|--------|---------|-----------|
| Route | `routes/playlists.py` → `get_songs(playlist_id)` (lines 34–40) | Receive the HTTP request, call the service, wrap the result as `{"songs": ..., "count": len(...)}`, map `ValueError` → 404 | `playlist_id` from URL | JSON response | Relevant as entry point; contains no bug — it faithfully returns whatever the service gives it (so an off-by-one in the service flows straight through to `count`). |
| Service | `services/playlist_service.py` → `get_playlist_songs(playlist_id)` (lines 38–66) | Validate the playlist exists, fetch its songs ordered by position, serialize to dicts | `playlist_id: str` | `list[dict]` of songs | **Relevant — contains the bug.** |
| — validation | line 53–55: `db.session.get(Playlist, playlist_id)` | Raise `ValueError` if the playlist doesn't exist | `playlist_id` | `Playlist` or raises | Not the cause; behaves correctly. |
| — query | lines 58–64: `query(Song).join(playlist_entries…).filter(playlist_id).order_by(position).all()` | Retrieve all songs in the playlist, ordered ascending by `position` | `playlist_id` | full ordered `list[Song]` (verified: 7 rows for a 7-entry playlist) | Not the cause; **verified correct** — returns every entry in order. |
| — return | line 66: `return [song.to_dict() for song in songs[:-1]]` | Serialize the songs to dicts | full `list[Song]` | dicts for all songs **except the last** | **Root cause.** |
| Helpers | `Song.to_dict()` (`models.py` 92–103) | Serialize a Song to a dict | `Song` | `dict` | Not the cause; serialization is fine. |
| Models / DB | `Song`, `playlist_entries` (`models.py`) | Storage + schema | — | rows | Not the cause; data is intact (DB has all 7 entries). |

## 3. Root Cause Analysis

### Issue Number and Title
**Issue #5 — The last song in a playlist never shows up.**

### How I Reproduced It
- Seeded the DB (`python seed_data.py`) and ran `pytest tests/test_playlists.py` — `test_playlist_returns_all_songs` (expected 5, got 4) and `test_playlist_returns_songs_in_order` (expected `[Track 1..Track 5]`, got `[Track 1..Track 4]`) both failed.
- Independently, calling `get_playlist_songs` for each seeded playlist returned 6 songs while the DB held 7 `playlist_entries` rows — the song at the highest `position` was always missing.

### How I Found the Root Cause
Tracing from the route, the only logic in the path is `get_playlist_songs`. I split that function into its two parts — the query and the return expression — and ran the query in isolation against the "Late Night Vibes" playlist. It returned all 7 songs in correct order, confirming the database and the query are correct. Manually applying the function's `songs[:-1]` slice to that result reproduced the exact 6-song output, pinpointing the slice as the sole cause.

### The Root Cause
**File:** `services/playlist_service.py` · **Function:** `get_playlist_songs` · **Line 66.**

The return statement was:

```python
return [song.to_dict() for song in songs[:-1]]
```

The query builds `songs` as the complete, position-ordered list of every song in the playlist. The Python slice `songs[:-1]` means "every element except the last one." So the function deliberately discards the final list element — the song at the highest `position` — before serializing.

- **What the code assumed:** nothing about the data required dropping an element; the function's own docstring states *"This function returns all songs in the playlist."*
- **What actually happens:** `[:-1]` removes the last item, so the function returns `n − 1` songs for an `n`-song playlist.
- **Why they differ:** the slice contradicts the intended (and documented) behavior — it is an off-by-one truncation applied to an otherwise-correct result set.
- **Why it produces the observed behavior:** because the query orders by `position` ascending, the dropped element is always the last/highest-position song — exactly matching the user report that "the last song never shows up." For a single-song playlist the effect is even more severe: `[:-1]` yields an empty list, hiding the only song.

### My Fix
Remove the slice so the comprehension iterates the full result set:

```python
# before
return [song.to_dict() for song in songs[:-1]]
# after
return [song.to_dict() for song in songs]
```

**Why this line is necessary and sufficient:** the query already returns the correct, fully-ordered set of songs, so the only defect is the truncation. Removing `[:-1]` makes the function return all songs, matching its docstring. No other line needs to change; the validation, query, ordering, and serialization were all already correct. Empty playlists remain safe (iterating an empty list yields `[]`, with no `IndexError`).

### Side-Effect Checks
- **Original reproduction (tests):** `pytest tests/test_playlists.py` — all 3 pass (previously 2 failed).
- **Full suite:** `pytest tests/` — 12 passed, 1 failed. The lone failure is `test_streak_increments_on_sunday` (Issue #1, untouched). Before the fix the suite had 3 failures (2 playlist + 1 streak); now only the unrelated streak failure remains, confirming the change is correctly scoped.
- **All seeded playlists:** each now returns all 7 entries including the last (`Free Throws`, `Harlem Renaissance`, `Lagos to London`).
- **Boundary — empty playlist:** returns `[]` with no error.
- **Boundary — single-song playlist:** now returns the 1 song (previously the slice would have returned `[]` — this was the most severe manifestation of the bug).
- **Neighboring functions in the same module:** `get_playlist` (metadata), `get_user_playlists`, and `create_playlist` all behave correctly and were not modified.
- **Ordering preserved:** returned songs remain in ascending `position` order (`test_playlist_returns_songs_in_order` passes).

## 4. Summary of Code Changes

| File | Change | Lines |
|------|--------|-------|
| `services/playlist_service.py` | In `get_playlist_songs`, changed `songs[:-1]` → `songs` in the return comprehension, so all songs are returned instead of all-but-last. | 1 line (line 66) |

No other files were modified. `submission.md` was updated with this documentation (deliverable, not source).

## 5. Side-Effect Testing Results

_See "Side-Effect Checks" above — summarized: 3/3 playlist tests pass; full suite 12 passed / 1 failed (only the unrelated Issue #1 streak test); all seeded playlists return complete lists; empty- and single-song boundaries behave correctly; neighboring playlist functions unaffected; ordering preserved._

## 6. Suggested Commit Message

_(Recommended only — not committed, per instructions.)_

```bash
git add services/playlist_service.py
git commit -m "fix: return all songs from get_playlist_songs instead of dropping the last"
```

---

_Milestone 3A complete for Issue #5. No other issues were investigated. Awaiting Milestone 3B instructions._

---
---

# Milestone 3B — Investigate, Fix, and Document Issue #1

_Scope: Issue #1 only (listening streak resets on Sundays). No other issues were investigated, and no unrelated code was changed._

## 1. Investigation Log

1. **Reviewed the Milestone 2 reproduction.** Confirmed: expected behavior is that listening on two consecutive calendar days increments the streak; actual behavior is that the streak resets to 1 when the second day is a Sunday; required condition is a Saturday→Sunday (more generally, *any day* → Sunday) transition; application state is a user with `last_listened_at` set to the prior day.
2. **Traced the execution path** from the entry point (`POST /songs/<song_id>/listen`, and the read path `GET /users/<user_id>/streak`). The streak-mutating logic lives entirely in `streak_service.update_listening_streak`.
3. **Read the implementation.** The branch on line 73 was `elif days_since_last == 1 and today.weekday() != 6:`. `weekday() == 6` is Sunday, so a consecutive-day listen whose current day is Sunday fails this branch and falls through to the reset `else`.
4. **Confirmed with an evidence sweep.** Ran `update_listening_streak` for every consecutive-day transition (Mon→Tue … Sun→Mon). Every transition incremented to 2 **except** the one where the second day is Sunday (Sat→Sun), which reset to 1. This isolated the defect to the `today.weekday() != 6` clause and ruled out any other cause.
5. **Compared against the documented rules.** The function's own docstring lists four rules (new user → 1; same day → no change; yesterday → +1; gap → reset) and says nothing about Sundays — confirming the weekday clause is an incorrect extra condition, not intended behavior.
6. **Used only throwaway investigation scripts** (run against an in-memory DB); no debug code was added to source files, so none needed removing.

## 2. Execution Trace

Write path: `POST /songs/<song_id>/listen`

| Step | Function / location | Responsibility | Inputs | Outputs | Relevant? |
|------|---------------------|----------------|--------|---------|-----------|
| Route | `routes/songs.py` → `listen(song_id)` (lines 43–53) | Parse JSON, require `user_id`, call the service, return the event; map `ValueError`→400 | `song_id`, `user_id` | JSON `ListeningEvent` | Entry point; no bug. |
| Service | `streak_service.record_listening_event(user_id, song_id)` (lines 14–39) | Look up the user, create a `ListeningEvent`, call `update_listening_streak(user, now)`, commit | `user_id`, `song_id` | `ListeningEvent` | Relevant — orchestrates the streak update, but the defect isn't here. |
| **Helper** | `streak_service.update_listening_streak(user, now)` (lines 42–78) | Apply the streak rules and mutate `user.listening_streak` / `user.last_listened_at` | `user`, `now` | mutates `user` | **Relevant — contains the bug (line 73).** |
| — new user | line 58–61 | streak = 1 if never listened | — | — | Correct. |
| — normalize tz | line 63–65 | make `last_listened_at` tz-aware | — | — | Correct; not the cause. |
| — day delta | line 67–68: `days_since_last = (today - last_date).days` | integer number of calendar days since last listen | dates | int | Correct; not the cause. |
| — same day | line 70–72: `if days_since_last == 0: return` | no change if already listened today | int | — | Correct. |
| — **increment** | line 73: `elif days_since_last == 1 and today.weekday() != 6:` | should increment on a consecutive day | int, weekday | branch taken or not | **Root cause** — the extra `and today.weekday() != 6` excludes Sundays. |
| — reset | line 75–76: `else: user.listening_streak = 1` | reset when a day was skipped | — | — | Correct in intent, but wrongly reached on Sundays because of line 73. |
| Models / DB | `User` (`models.py`) | persist `listening_streak`, `last_listened_at` | — | rows | Not the cause; storage is fine. |

Read path (`GET /users/<user_id>/streak` → `streak_service.get_streak`) merely returns the stored `listening_streak` and does not compute it, so it is not involved in the defect.

## 3. Root Cause Analysis

### Issue Number and Title
**Issue #1 — My listening streak keeps resetting.**

### How I Reproduced It
Ran `pytest tests/test_streaks.py`: `test_streak_increments_on_sunday` failed (`assert 1 == 2`) — it records a listen on Saturday 2024-06-15 then Sunday 2024-06-16 and expects the streak to reach 2, but it stayed at 1. The four other streak tests passed.

### How I Found the Root Cause
Tracing from the listen route, the only streak computation is in `update_listening_streak`. I swept `update_listening_streak` across all seven consecutive-day transitions with fixed dates. Every transition incremented to 2 except the one whose second day is Sunday (`weekday() == 6`), which reset to 1. That exactly matched the `and today.weekday() != 6` clause in the increment branch, pinpointing it as the cause. `Sun→Mon` incrementing correctly further confirmed the trigger is *the current day being Sunday*, not the previous day.

### The Root Cause
**File:** `services/streak_service.py` · **Function:** `update_listening_streak` · **Line 73.**

The branch read:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

- **What the code assumed / should do:** per its docstring, "If the user listened yesterday: streak increments by 1" — a consecutive-day listen should always increment, regardless of weekday.
- **What actually happens:** the extra condition `today.weekday() != 6` (Python's `weekday()` is Monday=0 … Sunday=6) makes the increment branch *false whenever the current day is a Sunday*. Execution then falls through to the `else`, which resets the streak to 1.
- **Why they differ:** the weekday clause is an incorrect additional requirement with no basis in the documented rules — it treats Sunday as if a day had been skipped.
- **Why it produces the observed behavior:** a user who listens every day has `days_since_last == 1` each day; on Sundays the increment is skipped and the streak resets to 1 — matching the report that the streak "keeps resetting." It recovers on Monday (weekday 0), so the symptom is a weekly reset.

### My Fix
Remove the erroneous weekday clause so a consecutive-day listen always increments:

```python
# before
elif days_since_last == 1 and today.weekday() != 6:
# after
elif days_since_last == 1:
```

**Why this change is necessary and sufficient:** the only defect is the extra `and today.weekday() != 6` condition; removing it restores the documented rule ("yesterday → +1") for all weekdays. The same-day branch (line 70) and the reset `else` (line 75, reached when `days_since_last > 1`) were already correct and are unaffected, so no other line needs to change.

### Side-Effect Checks
- **Original reproduction (tests):** `pytest tests/test_streaks.py` — all 5 pass (previously 1 failed).
- **Full suite:** `pytest tests/` — **13 passed, 0 failed** (Issue #5's playlist fix from 3A remains green; no regressions).
- **Consecutive-day increments across every weekday:** Mon→Tue … Sun→Mon all reach 2 (previously Sat→Sun reset to 1).
- **Same-day, no double-count:** morning+evening on the same day stays at 1 — verified for a weekday and for **Sunday** specifically.
- **Skipped-day resets still work:** Mon→Wed, **Fri→Sun (skip Sat)**, and **Sat→Mon (skip Sun)** all correctly reset to 1 — confirming the reset path is intact and the fix didn't over-correct.
- **Multi-day streak crossing a Sunday:** Fri→Sat→Sun→Mon now yields a streak of 4 (previously it would have broken at Sunday).

## 4. Code Change Summary

| File | Change | Purpose |
|------|--------|---------|
| `services/streak_service.py` | In `update_listening_streak`, removed `and today.weekday() != 6` from the consecutive-day branch (line 73). | Increment the streak on consecutive days for every weekday, including Sundays. |

No other files were modified. `submission.md` updated with this documentation (deliverable, not source).

## 5. Side-Effect Testing Results

_See "Side-Effect Checks" above — summarized: 5/5 streak tests pass; full suite 13 passed / 0 failed; consecutive-day increments verified for all 7 weekday transitions; same-day non-double-count verified (incl. Sunday); skipped-day resets verified around Sunday boundaries; multi-day streak crossing Sunday reaches the correct value._

## 6. Suggested Commit Message

_(Recommended only — not committed, per instructions.)_

```bash
git add services/streak_service.py
git commit -m "fix: increment listening streak on consecutive Sundays instead of resetting"
```

---

_Milestone 3B complete for Issue #1. No other issues were investigated. Awaiting instructions before starting Issue #4._

---
---

# Milestone 3C — Investigate, Fix, and Document Issue #4

_Scope: Issue #4 only (a notification is created when a friend adds your song to a playlist, but not when they rate it). No other issues were investigated, and no unrelated code was changed._

## 1. Investigation Log

1. **Reviewed the Milestone 2 reproduction.** Confirmed: expected behavior is that rating another user's song notifies the song's sharer (mirroring the playlist-add notification); actual behavior is that no notification is created; required state is a song shared by one user and a *different* user submitting a rating.
2. **Traced the execution path** from the endpoint used in reproduction (`POST /songs/<song_id>/rate`) into `notification_service.rate_song`.
3. **Read `rate_song` and compared it against the working `add_to_playlist`.** `add_to_playlist` (lines 64–70) ends by calling `create_notification(...)` for `song.shared_by` when the actor isn't the sharer. `rate_song` (lines 73–110) validates the score, upserts the `Rating`, commits, and returns — **with no call to `create_notification` anywhere**.
4. **Noted corroborating evidence in the code itself.** `create_notification`'s docstring lists `'song_rated'` as an example `notification_type` (line 19), indicating a rating notification was intended but never wired up.
5. **Reproduced with instrumentation.** Rating "Crown Heights Anthem" (shared by `simone`) as `nova`: the `Rating` count went 0→1 and `rate_song` returned a valid `Rating`, while `simone`'s notification count stayed 0→0 — proving the rating path works and only the notification is missing.
6. **Used only throwaway scripts** against the seeded DB; no debug code was added to source, so none needed removal.

## 2. Execution Trace

`POST /songs/<song_id>/rate`

| Step | Function / location | Responsibility | Inputs | Outputs | Relevant? |
|------|---------------------|----------------|--------|---------|-----------|
| Route | `routes/songs.py` → `rate(song_id)` (lines 29–40) | Parse JSON, require `user_id` and `score`, call the service, return `rating.to_dict()` (201); map `ValueError`→400 | `song_id`, `user_id`, `score` | JSON `Rating` | Entry point; no bug. |
| Service | `notification_service.rate_song(user_id, song_id, score)` (lines 73–110) | Validate score range, look up song & rater, upsert the `Rating`, commit | `user_id`, `song_id`, `score` | `Rating` | **Relevant — the notification step is missing here.** |
| — validation | line 85–86 | reject score outside 1–5 | `score` | raises `ValueError` | Correct; not the cause. |
| — lookups | line 88–94 | fetch `Song` and rater `User`, 404-style `ValueError` if missing | ids | `Song`, `User` | Correct; also make `song`/`rater` available for a notification. |
| — upsert | line 96–108 | update existing rating or create a new one, then commit | — | `Rating` | Correct; the rating is persisted. |
| — **return** | line 110: `return rating` | return the rating | — | `Rating` | **Root cause: the function returns without ever creating a notification.** |
| Helper (not called) | `create_notification(user_id, type, body)` (lines 13–32) | persist a `Notification` for a user | — | `Notification` | Exists and works (used by `add_to_playlist`) — but `rate_song` never calls it. |
| Reference (working) | `add_to_playlist` (lines 64–70) | notifies `song.shared_by` when actor ≠ sharer | — | — | The correct pattern that `rate_song` should mirror. |
| Models / DB | `Rating`, `Notification` (`models.py`) | storage | — | rows | Not the cause. |

## 3. Root Cause Analysis

### Issue Number and Title
**Issue #4 — I got notified when a friend added my song to a playlist, but not when they rated it.**

### How I Reproduced It
Seeded the DB and, as `nova`, rated `simone`'s song "Crown Heights Anthem". The rating was created (rating count 0→1, a valid `Rating` returned), but `simone`'s notification count stayed at 0 — no notification was generated.

### How I Found the Root Cause
Tracing from `POST /songs/<song_id>/rate`, the only service is `rate_song`. Reading it end to end shows it validates, upserts the rating, commits, and returns — with no notification logic. Comparing with the sibling `add_to_playlist`, which ends by calling `create_notification` for the song's sharer, made the omission explicit: the two "someone interacted with your song" paths were implemented inconsistently. The reproduction confirmed the divergence (rating persists; notification absent).

### The Root Cause
**File:** `services/notification_service.py` · **Function:** `rate_song` · **missing logic before line 110 (`return rating`).**

`rate_song` never calls `create_notification`. The mechanism for notifying users exists (`create_notification`) and is used by `add_to_playlist`, but the rating path was never wired to it.

- **What the code should do:** like `add_to_playlist`, notify the song's original sharer when someone *else* interacts with their song (here, rates it). The `create_notification` docstring even anticipates a `'song_rated'` type.
- **What actually happens:** `rate_song` saves the rating and returns; no `Notification` row is ever created.
- **Why they differ:** the notification call is simply absent from `rate_song` — an omission, not a wrong condition.
- **Why it produces the observed behavior:** with no `create_notification` call, the sharer's notification list is unchanged after a rating, exactly matching the report ("notified on playlist-add but not on rating").

### My Fix
Add the same notify-the-sharer step used by `add_to_playlist`, after the rating is committed:

```python
    db.session.commit()

    # Notify the person who originally shared the song (if it wasn't them who rated it)
    if song.shared_by != user_id:
        create_notification(
            user_id=song.shared_by,
            notification_type="song_rated",
            body=f"{rater.username} rated your song '{song.title}' {score}/5.",
        )

    return rating
```

**Why each line is necessary:** the `if song.shared_by != user_id` guard mirrors `add_to_playlist` so a user isn't notified for rating their own song; `create_notification(...)` is the existing, tested mechanism for persisting a notification; `notification_type="song_rated"` matches the type documented in `create_notification`; the `body` follows the human-readable style of the playlist-add message. It is placed *after* `db.session.commit()` so the notification is only created once the rating has successfully persisted, and *before* `return rating` so the return value is unchanged. `song` and `rater` are already loaded earlier in the function, so no extra queries are needed. No existing line is modified.

### Side-Effect Checks
- **Original reproduction:** rating another user's song now creates exactly one notification for the sharer — type `song_rated`, body `"nova rated your song 'Crown Heights Anthem' 5/5."` (count 0→1).
- **Full suite:** `pytest tests/` — **13 passed, 0 failed** (Issues #5 and #1 remain green; no regressions).
- **Rating still persists correctly:** the `Rating` row is created with the correct score.
- **Duplicate rating:** re-rating by the same user still updates the existing `Rating` in place (one row, score changed) — consistent with prior behavior. It also produces another `song_rated` notification, which is consistent with `add_to_playlist` (both notify on each qualifying call rather than deduplicating).
- **Self-rating:** a user rating their **own** song creates **no** notification (guard works; delta 0).
- **Playlist-add notifications still work:** unchanged — adding a song to a playlist still notifies the sharer (delta 1).
- **Invalid score:** `score=9` still raises `ValueError` before any notification is created (delta 0), so validation is unaffected.

## 4. Code Change Summary

| File | Change | Purpose |
|------|--------|---------|
| `services/notification_service.py` | In `rate_song`, added a `create_notification(...)` call (guarded by `song.shared_by != user_id`) after the commit, before the return. | Notify the song's original sharer when a *different* user rates their song, matching the existing playlist-add behavior. |

No other files were modified. `submission.md` updated with this documentation (deliverable, not source).

## 5. Side-Effect Testing Results

_See "Side-Effect Checks" above — summarized: full suite 13 passed / 0 failed; rating persists and upserts correctly; self-rating does not notify; playlist-add notifications still work; invalid scores still rejected with no notification; the new `song_rated` notification is created for the sharer with the correct type and body._

## 6. Suggested Commit Message

_(Recommended only — not committed, per instructions.)_

```bash
git add services/notification_service.py
git commit -m "fix: notify song owner when another user rates their song"
```

---

_Milestone 3C complete for Issue #4. Three issues (#5, #1, #4) have now been investigated, fixed, verified, and documented._
