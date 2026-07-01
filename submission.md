# Mixtape Codebase Map

Mixtape is a Flask app for sharing songs with friends, building collaborative playlists, and tracking listening stats (streaks, activity feeds, notifications). It uses Flask-SQLAlchemy with a SQLite database by default and organizes code into three layers: routes (HTTP), services (business logic), and models (persistence).

## Main Files

### app.py
Application factory. `create_app(config)` builds the Flask app, sets default config (SQLite URL from `DATABASE_URL` env var, falling back to `sqlite:///mixtape.db`), initializes the shared `db = SQLAlchemy()` instance, and registers four blueprints with URL prefixes: `songs_bp` at `/songs`, `playlists_bp` at `/playlists`, `users_bp` at `/users`, `feed_bp` at `/feed`. It calls `db.create_all()` inside an app context at startup, so there is no migration system, schema changes just require a fresh database. When run directly, it creates the app and calls `.run(debug=True)`.

### models.py
Defines 7 SQLAlchemy models plus 3 association tables, all using UUID string primary keys (`generate_uuid()`), not autoincrementing integers.

**Models**: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`.

**Association tables**:
- `friendships`: self-referential many-to-many on `User`.
- `song_tags`: many-to-many between `Song` and `Tag`.
- `playlist_entries`: many-to-many between `Playlist` and `Song`, but it is not a pure join table. It carries extra columns (`position`, `added_by`, `added_at`), meaning songs in a playlist have an explicit order and an audit trail of who added them, not just insertion order.

Every model defines `to_dict()` for direct use in `jsonify()`. There is no separate `PlaylistSong` model class, the ordering data lives only in the raw `playlist_entries` Table object, so any code that needs `position` has to query the table directly (see `playlist_service.get_playlist_songs`) rather than going through the ORM relationship.

There is no dedicated rating model exposed as a "review", `Rating` is a lightweight score (1-5) with a `UniqueConstraint("user_id", "song_id")`, so a user can only have one rating per song. Later ratings update the existing row, an upsert implemented in the service layer, not the database.

### services/
Each module owns one feature area and is the only place that talks to `db.session` for that feature.

- **feed_service.py**: `get_friends_listening_now(user_id)` returns each friend's most recent song listened to within the last 24 hours, deduplicated to one entry per friend. `get_activity_feed(user_id, limit=20)` returns the most recent N listening events from friends regardless of recency (no time cutoff). Both raise `ValueError` if the user doesn't exist and return `[]` if the user has no friends.
- **notification_service.py**: `create_notification()` is the single low-level constructor other notification producing functions call. `add_to_playlist()` adds a song to a playlist's `songs` relationship (skipping if already present) and notifies the song's original sharer, unless the sharer is the one who added it. `rate_song()` performs an upsert on `Rating` by looking up an existing `(user_id, song_id)` row before inserting. `get_notifications()` and `mark_as_read()` handle retrieval and read state.
- **playlist_service.py**: `create_playlist()`, `get_playlist()`, `get_user_playlists()` are straightforward CRUD/query wrappers. `get_playlist_songs()` joins `Song` to the raw `playlist_entries` table to sort by `position`.
- **search_service.py**: `search_songs(query)` does a case-insensitive `ilike` match on title or artist, joined against `song_tags`. `get_song(song_id)` is a single lookup.
- **streak_service.py**: `record_listening_event()` inserts a `ListeningEvent` and calls `update_listening_streak()` in the same transaction. The streak logic is calendar day based (UTC): same day is a no-op, exactly one day later increments, anything else resets to 1.

### routes/
Four blueprint modules (`songs.py`, `playlists.py`, `users.py`, `feed.py`). Every route is a thin wrapper: parse `request.args` or `request.get_json()`, call exactly one service function, catch `ValueError` and translate it into a `(jsonify({"error": ...}), 4xx)` response. No route contains business logic or direct `db.session` calls except `users.py`'s `get_user`, which queries `User` directly instead of going through a service, the one inconsistency in an otherwise strict layering.

### tests/
Pytest suite covering three services: `test_playlists.py`, `test_search.py`, `test_streaks.py`. Each uses a `create_app` fixture pointed at `sqlite:///:memory:`. Several tests contain comments naming an expected bug and its symptom right above the assertion, e.g. `assert len(songs) == 5  # Bug causes this to return 4`. I ran the suite (`pytest tests/ -q`): 10 pass, 3 fail, confirming those comments describe real, currently failing behavior rather than resolved history.

## Data Flow: Rating a Song

`POST /songs/<song_id>/rate` in [routes/songs.py](routes/songs.py) extracts `user_id` and `score` from the JSON body and calls `rate_song(user_id, song_id, int(score))` in `notification_service.py`. That function validates the score is 1-5, checks for an existing `Rating` row for that `(user_id, song_id)` pair, and either updates its `score` or inserts a new `Rating`, then commits. It returns the `Rating` row, which the route serializes with `to_dict()` and returns as `201`.

**Notable gap**: despite living in `notification_service.py` alongside `add_to_playlist()` (which does call `create_notification`), `rate_song()` never calls `create_notification()`. Rating a song does not notify the original sharer, unlike the playlist add flow below. This is Issue 4 in the tracked bug list (see below), it is the module's one write path that skips the notification step it clearly should mirror.

## Data Flow: Adding a Song to a Playlist (the fully wired notification example)

`POST /playlists/<playlist_id>/songs` in [routes/playlists.py](routes/playlists.py) reads `song_id` and `added_by` from the body and calls `add_to_playlist(playlist_id, song_id, added_by)`. That function:
1. Loads the `Song`, adder `User`, and `Playlist`, raising `ValueError` (mapped to 400) if any are missing.
2. Appends the song to `playlist.songs` (the ORM relationship backed by `playlist_entries`) if not already present, and commits.
3. If the adder is not the song's original sharer, calls `create_notification(user_id=song.shared_by, notification_type="song_added_to_playlist", body=...)`, which inserts a `Notification` row and commits separately.

The recipient later fetches this via `GET /users/<user_id>/notifications` in [routes/users.py](routes/users.py), which calls `get_notifications()`.

**Gap worth knowing**: step 2's `playlist.songs.append(song)` goes through the ORM many-to-many relationship, which never sets a `position` value (SQLAlchemy just inserts the two foreign keys into `playlist_entries`, `position` would violate `nullable=False` unless a database default exists, and none does here). Meanwhile `get_playlist_songs()`, the read path, orders and slices by `playlist_entries.c.position`. A song added through this route sits on a code path the read side wasn't built to expect. This asymmetry between the write path (ORM relationship) and read path (raw table columns) is confirmed by the two failing playlist tests below.

## Data Flow: Listening to a Song (streaks + feed)

`POST /songs/<song_id>/listen` calls `record_listening_event(user_id, song_id)`, which inserts a `ListeningEvent` row and updates `user.listening_streak` / `last_listened_at` in the same commit. This is the only writer of `ListeningEvent`, and both `feed_service` functions are pure readers of that table, joined against the `friendships` association to scope results to the caller's friends.

## Patterns Noticed

**Strict route-to-service delegation**: with one exception (`users.get_user`), every route parses input, calls one service function, and formats the response. All persistence and business rules live in `services/`. This makes the services layer the right place to write new tests or reason about correctness, routes are barely worth testing beyond status codes.

**Service functions own their own commits**: there's no unit of work or transaction wrapper spanning multiple service calls. Each service function calls `db.session.commit()` itself. `add_to_playlist()` actually commits twice: once for the playlist append, once inside `create_notification()`. A failure partway through (e.g., the notification insert) would not roll back the playlist change. Consistent, but means composing two service calls in a new feature risks partial writes.

**Validation via exceptions, not a schema layer**: there's no request validation library. Every service raises a plain `ValueError` with a human-readable message for both "not found" and "bad input" cases, and every route catches `ValueError` and maps it to 400 or 404 by convention. The exception message becomes the API's error string, so wording changes in a service are a de facto API contract change.

**Raw association tables used directly when relationships aren't enough**: the ORM `secondary=` relationships (`Song.tags`, `Playlist.songs`, `User.friends`) are used for simple membership, but as soon as extra columns or ordering are needed (`playlist_entries.position`), code drops to querying the raw `Table` object with explicit joins instead of modeling it as a first class entity. Deliberate simplicity trade-off, but it's also why the playlist add and playlist read paths drift out of sync (see gap above): the write path uses the high-level relationship while the read path depends on a column the write path doesn't set.

**UUID primary keys everywhere**: every model uses a string UUID default rather than an integer autoincrement ID, generated in application code (`generate_uuid()`), not the database.

## The Five Tracked Issues (per README.md)

This repo is the "Project 5: Mixtape Bug Hunt" starter, README.md lists five open issues, three of which are wired to failing tests and confirmed by running `pytest tests/ -q` (10 passed, 3 failed):

1. **"My listening streak keeps resetting"**, `streak_service.py`. Confirmed: [services/streak_service.py:73](services/streak_service.py#L73) only increments a one day gap `if days_since_last == 1 and today.weekday() != 6`, explicitly excluding Sunday, so listening Saturday then Sunday resets the streak to 1 instead of incrementing to 2. Fails `test_streaks.py::test_streak_increments_on_sunday` (`assert 1 == 2`). The `!= 6` guard reads like a leftover from unrelated "skip weekends" logic that doesn't belong in a streak feature.
2. **"Friends Listening Now shows people from yesterday"**, `feed_service.py`. Not covered by a test, so I read the code directly: `get_friends_listening_now()` uses a rolling `RECENT_THRESHOLD = timedelta(hours=24)` window, not a calendar day boundary. A friend who listened at 11:58pm yesterday still shows up at 12:01am today because only 3 minutes have elapsed, even though it's a different calendar day. Whether this is "the bug" depends on the intended semantics (rolling window vs. calendar day), worth confirming against the full issue description in the project brief before fixing.
3. **"The same song keeps showing up twice in search"**, `search_service.py`. I initially suspected this from `search_songs()`'s `outerjoin(song_tags, ...)`, since a raw SQL join does produce one row per tag. But `test_search.py`'s three duplicate related tests all pass: `db.session.query(Song)...all()` returns ORM identity-mapped `Song` objects, and SQLAlchemy's ORM layer collapses duplicate rows for the same entity even though the underlying SQL produces multiples. So this specific query is not actually buggy today, worth flagging as a false lead if the brief describes different repro steps.
4. **"I got notified when a friend added my song to a playlist but not when they rated it"**, `notification_service.py`. Confirmed by reading: `add_to_playlist()` calls `create_notification()` when someone else's shared song is added to a playlist, but `rate_song()` never calls it. Not covered by a failing test, easy to miss if you only run the suite.
5. **"The last song in a playlist never shows up"**, `playlist_service.py`. Confirmed: [services/playlist_service.py:66](services/playlist_service.py#L66) does `return [song.to_dict() for song in songs[:-1]]` after the ordered query, silently truncating the final entry regardless of playlist size. Fails `test_playlists.py::test_playlist_returns_all_songs` (`assert 4 == 5`) and `test_playlist_returns_songs_in_order` (Track 5 missing).

Net: issues 1, 4, and 5 are solidly confirmed (two by failing tests, one by direct reading with no notification call anywhere in the rate path). Issue 3 does not reproduce against the current test suite and needs the brief's exact repro steps to pin down. Issue 2 needs the brief's definition of "recent" to know if the 24 hour rolling window is the actual defect.

---

## AI Usage

This map was drafted independently by reading the actual source files (models.py, app.py, every file in services/ and routes/, the test suite, and README.md) and running `pytest tests/ -q` to verify claims empirically rather than trust docstrings or comments. It was then refined with AI assistance for structure and completeness, not the other way around. Below is my original, unrefined submission as pasted to the assistant, followed by the assistant's evaluation of it against the finished map above.

### My Original Notes (as submitted, unedited)

The app is an overall fundamental yet comprehensive user song application for mixtapes where friends can share songs, built collab playlists, and track listening stats.

 model.py defines the main classes utilized in the repository, including User, Tag, Song, ListeningEvent, Rating, Playlist, and Notification. In addition, it also defines the relational schema via SQLAlchemy & for all the aforementioned respective classes initializes the columns/features that each SQL table for the classes will have. Each also includes a to_dict() method used for returning selected attributes from the class tables as dict, which allows easy jsonify() formatting via Flask. 
 
 app.py provides the Flask structure for launching the app, including default configuration such as the database URL. It registers blueprints via url_prefix and returns the app object in the create_app() function, when the module is run it creates this by calling the aforementioned function & running the app with debug mode on. 
 
 The services directory includes modules for feed_service.py (Handles "Friends Listening Now" feed & activity logic) which gets the friends listening now & acitvity feed of the user. notification_service.py handles creating & retrieving notifications, with functions for creating a notification object and committing it to the db, adding it to playlist, rating a song, getting notifications, and marking as read. playlist_service handles playlist creation & retrieval logic, including creating a new playlist and adding it to the db, getting songs in the playlist via db join, getting playlist, and getting user playlists. search_service.py handles song search logic, including searching for songs which goes by title or artist name via query, and getting a single song by ID. streak_service.py handles listening logic for users such as recording an event via the datetime module TTL timestamped, update the listening streak, getting streak etc. 
 
 The routes directory covers feeding routes for the Flask app including for mixtape, mixtape playlists, songs, and users with several interface level functions in each module that return jsonify-ed objects. 
 
 The tests directory includes a fundamental suite with unit tests for playlists, search, and streaks via pytest for easy launching of the suite. Data flow — user rates a song: POST /songs/<id>/rate in routes/songs.py calls notification_service.notify_song_rated(). That function creates a Notification record for the song's original sharer. There's no separate rating model — the rating is stored directly on the Song.

### Assistant's Evaluation (judging the above against the actual codebase)

What was accurate: the file-by-file, function-by-function summary of app.py, models.py, and each services/ module was correct at the level of "what does this do." Identifying that there's no separate rating model, and using the rating flow as the worked data flow example, matched the assignment's own template for a strong map.

What was missing, and what the refined map above adds:

1. **The described data flow was factually wrong.** The notes claim `POST /songs/<id>/rate` calls a function named `notify_song_rated()` that creates a notification for the song's sharer. No such function exists anywhere in the codebase. The actual route calls `notification_service.rate_song()`, which only writes to the `Rating` table and never creates a `Notification`. This is is not a paraphrase error, it is the exact missing behavior tracked as Issue 4 in README.md ("I got notified when a friend added my song to a playlist but not when they rated it"). The original notes describe the feature working as if the bug were already fixed.
2. **No verification against the code or tests.** The notes never mention running `pytest`, never surface that 3 of 13 tests currently fail, and never mention that some test files contain comments describing known bugs. The finished map treats every claim as something to verify by execution, the original notes treat the code's apparent structure as ground truth.
3. **README.md was never referenced.** The repo is explicitly framed as "Project 5: Mixtape Bug Hunt" with five named, tracked issues. The original notes reconstruct roughly the same problem space from code alone but never checks whether documentation of known issues already exists.
4. **No mention of the two other confirmed bugs** (the Sunday streak-reset condition, the `[:-1]` slice dropping the last playlist song) or the architectural gap between the playlist write path (ORM relationship, no `position` set) and read path (raw table query ordered by `position`).
5. **Minor layering inconsistency missed**: `routes/users.py`'s `get_user` bypasses the service layer and queries the model directly, breaking the otherwise consistent routes-delegate-to-services pattern the notes correctly identified elsewhere.

Overall: strong grasp of static structure, but the one dynamic claim in the original notes (the rating notification flow) was incorrect, and it happened to be incorrect in exactly the way the assignment was designed to test.
