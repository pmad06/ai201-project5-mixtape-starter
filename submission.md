## AI Usage

Throughout this project, I used Claude for codebase navigation, understanding functions that had errors within them, and working through the debugging logic. 

**Codebase orientation**: In order to understand all of the files quickly and thoroughly, I gave Claude the app's main files and asked it to summarize what each file consisted of and their overall relevance to the project. Rather then reading every file one by one, this was more helpful because I was able to conceptually understand the code rather than just understanding what each line does. 

**Where AI helped correctly**: Claude was able to help for both issues #1 and #4. For Issue #1, Claude helped me understand the function where the incorrect elif statement was found and how exactly the streak was resetting. Once it explained what exactly the problem was within that function, it was easy to figure out how to solve the issue and how I needed to edit the code to do so. For Issue #4, Claude helped me figure out which functions I need to look into to figure out why the issue was occuring and the fix became a lot easier after understanding what was happening conceptually. 

**Where AI was wrong**: Claude did not fully understand the code and was trying to come up with solutions for Issue #3 without actually testing it. Although it had all of the main files, the AI did not have the ability to actually test the code and determine whether it was right or not. However, after testing two different seeded songs, the results showed that neither one produced duplicate results. So, although the AI was trying to suggest a solution from simply reading the code, I dropped the issue instead and started working on the next one to save time and make more progress. 

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

# Plan
Tackling: streak reset (#1), no notification (#4), playlist last-song bug (#5) first 

## Root Cause Analysis 

# Issue #1 - Listening streak keeps resetting 

**How I reproduced it**: Opened a flash shell and called update_listening_streak() on a seeded user. Set the user's listening streak to 3 and last_listened_at to a confirmed Saturday, then called the function with "now" set to the next day. Confirmed sunday.weekend() printed 6 which resulted in the streak going from 3 to 1, instead of increasing. Ran an identical scenario with "now" set to a Monday instead, where the streak went from 3 to 4. So, only Sunday produced the wrong result. 

**How I found the root cause**: Read update_listening_streak() inside services/streak_service.py. The branch that needed to be edited:

elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1

Since today.weekday() returned 6 for Sunday, today.weekday() != 6 evaluates to False on a Sunday. Traced the boolean logic: with
days_since_last == 1 (True) and today.weekday() != 6 (False) on a
Sunday, True and False evaluates to False, so the elif doesn't fire
and execution falls through to else, resetting the streak to 1.

**The root cause**: The first elif statement has an extra condition, today.weekday() != 6, which excludes Sunday from incrementing the streak. Since the condition is written as an exclusion rather than a special case, it disables the increment entirely on Sundays, which results in the else statement to be true and the streak resetting to 1 instead. 

**My fix and side-effect check**: Remove the today.weekday() != 6 clause from the elif statement. This matched the documented rule, which was a one-day gap always increments, regardless of a weekday. Re-ran the Sunday reproduction test after the fix and streak went from 3 to 4. Made sure there were no side effects by re-running the Monday case, as the streak incremented from 3 to 4. Updated branch: 

elif days_since_last == 1:
    user.listening_streak += 1


# Issue #4 - No Notification when a friend rates a shared song 

**How I reproduced it**: In flask shell, picked a seeded song and its
original sharer, then found a different user to act as the rater. Called
get_notifications(sharer_id) before rating and got 1 existing notification
(a playlist-add notification planted by seed data). Then called
rate_song(rater.id, song.id, 5), which succeeded and returned a valid
Rating object. Called get_notifications(sharer_id) again but only 1
notification, unchanged. The rating saved correctly, but no notification
was ever created for the sharer.

**How I found the root cause**: Opened services/notification_service.py
and compared rate_song() line-by-line against add_to_playlist() in the
same file, since the two functions handle structurally similar situations
(a user interacts with someone else's shared song). add_to_playlist()
ends with a guarded call to create_notification():

if song.shared_by != added_by_user_id:
    create_notification(...)

**The root cause**: rate_song() was never wired up to the notification
system. The rating logic itself is
correct and complete, but the function is simply missing the
check-and-notify step that its sibling function (add_to_playlist())
already implements for a different action. The pattern exists in the
codebase; it was just never applied here.

**My fix and side-effect check**: Added the same guarded
create_notification() call used in add_to_playlist(), adapted to
rate_song()'s own variables:

if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score} stars.",
    )

Re-ran the reproduction test after the fix: notification count went from
1 → 2, and the new entry read exactly as expected: song_rated - darius rated your song 'Midnight Drive' 5 stars. Confirmed the existing
song_added_to_playlist notification was untouched.

# Issue #5 - Last song in a playlist never shows up 

**How I reproduced it**: Checked seed_data.py to confirm how many songs
were seeded per playlist, whereeach of the 3 seeded playlists gets 7 songs
(e.g. all_songs[:7] for "Late Night Vibes"). In flask shell, called
get_playlist_songs() directly on that playlist's ID. Expected 7 songs
back; got 6. Printed the titles — the 7th (last-added) song was missing
from the result every time.

**How I found the root cause**: Opened services/playlist_service.py
and read get_playlist_songs(). The query itself correctly joins
playlist_entries and orders by position. The bug was in the return
statement:

return [song.to_dict() for song in songs[:-1]]

**The root cause**: songs[:-1] is a Python slice meaning "all but the
last item." The slicing
happens on the raw query results before formatting, silently discarding
whichever song is last in position order, regardless of playlist size.

**My fix and side-effect check**: Changed the return statement: 

return [song.to_dict() for song in songs]

