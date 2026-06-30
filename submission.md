## Codebase Map 

# Main Files 
- app.py: Flask application factory (create_app()). Initializes db = SQLAlchemy(), sets config(DB URI, secret key), registers the following blueprints: songs, playlists, users, feed, and calls db.create_all() on startup. 
- models.py: defines all SQLAlchemy models: User, Tag, Song, ListeningEvent, Rating, Playlist, Notification. Defines three association tables: friendships, song_tags, playlist_entries. 
- routes/songs.py: /songs endpoints: search, get detail, rate a song, log a listening event. Delegates to search_service and notification_service, and streak_service.
- routes/playlists.py: /playlists endpoints: create playlist, get playlist detail, get playlist songs, add a song to a playlist. Delegates to playlist_service and notification_service.add_to_playlist.
- seed_data.py: builds a fresh DB: 5 users with a friendship graph, 25 songs, 3 playlists of 5-7 songs each, a mix of recent and older listening events, manually-set last_listened_at values for streak testing, and one working "song added to playlist" notification. 
- routes/users.py: /users endpoints: get user, get streak, get notifications, mark notification read. Delegates to streak_service and notification_service.
- routes/feed.py: /feed endpoints: friends listening now, activity feed. Delegates to feed_service.
- services/search_service.py: search_songs() queries Song joined to song_tags, filters by title/artist. get_song() fetches one song by ID.
- services/playlist_service.py: create_playlist() creates a Playlist. get_playlist_songs() queries songs ordered by position in playlist_entries.
- services/notification_service.py: create_notification() writes a Notification row. add_to_playlist() adds a song and notifies the sharer. rate_song() creates/updates a Rating.
- services/feed_service.py: get_friends_listening_now() finds friends' recent ListeningEvents within a 24hr window. get_activity_feed() is similar but not time-filtered.
- services/streak_service.py: record_listening_event() logs an event and updates the streak based on days since last listen.


# Data Flow: Rating a Song -> Notification
1. Client sends POST /songs/<song_id>/rate with user_id and score in the JSON body. 
2. routes/songs.py::rate() validates both fields are present, calls notification_service.rate_song(user_id, song_id, int(score)).
3. rate_song() validates the score is 1-5, finds-or-updates the Rating row, commits, and returns it. 
4. rate_song() never calls create_notification(). Compare to add_to_playlist() in the same file, which explicity notifies the song's original sharer after the playlist write. rate_song() has no equivalent call.
5. Route returns the serialized Rating (via .to_dict()) with a 201.

# Patterns Noticed 
- Routes only parse requests and format responses — all business logic lives in services/.
- Every model uses a UUID string as primary key, not autoincrementing ints.
- playlist_entries is a richer join table (has position, added_by, added_at) compared to the plain friendships/song_tags tables.