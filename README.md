# Mixtape Debugging

Mixtape is a Flask-based social music application where friends can share songs, build collaborative playlists, rate music, track listening streaks, and receive activity notifications.

This repository focuses on **debugging and maintaining an existing codebase**. I investigated reported issues in the application's service layer, identified their root causes, implemented fixes, and verified the behavior through automated and manual testing.

---

## What I Worked On

I fixed four bugs affecting different parts of the application:

* Listening streaks incorrectly resetting on Sundays
* Friends appearing as "Listening Now" hours after they stopped listening
* Missing notifications when another user rated a shared song
* The final song in a playlist being dropped from results

Each fix was developed separately and committed using a dedicated `fix:` commit.

---

## Tech Stack

* **Python**
* **Flask**
* **SQLAlchemy**
* **SQLite**
* **pytest**

---

## App Structure

```text
mixtape-debugging/
├── app.py                      # Flask app factory and database setup
├── models.py                   # SQLAlchemy models
├── routes/
│   ├── songs.py                # Song sharing, search, rating, and listening
│   ├── playlists.py            # Playlist creation and song management
│   ├── users.py                # User profiles, streaks, and notifications
│   └── feed.py                 # Listening and activity feeds
├── services/
│   ├── streak_service.py       # Listening streak logic
│   ├── feed_service.py         # Friends Listening Now logic
│   ├── search_service.py       # Song search logic
│   ├── notification_service.py # Ratings and notification logic
│   └── playlist_service.py     # Playlist retrieval logic
├── tests/
│   ├── test_streaks.py
│   ├── test_search.py
│   └── test_playlists.py
├── seed_data.py                # Populates the database with test data
├── submission.md               # Detailed debugging and root-cause analysis
├── requirements.txt
└── .gitignore
```

The application is organized into three main layers:

```text
Browser → Routes → Services → Models → Database
```

* **Routes** receive and validate requests.
* **Services** contain the application's business logic.
* **Models** define the structure and relationships of stored data.
* **SQLite** provides persistent local storage.

Most of the debugging work in this project focused on the `services/` layer.

---

## Setup

### Create a Virtual Environment

```bash
python -m venv .venv
```

Activate it:

### macOS / Linux

```bash
source .venv/bin/activate
```

### Windows Command Prompt

```bash
.venv\Scripts\activate.bat
```

### Windows Git Bash

```bash
source .venv/Scripts/activate
```

### Install Dependencies

```bash
pip install -r requirements.txt
```

### Seed the Database

```bash
python seed_data.py
```

### Run the Application

```bash
FLASK_APP=app:create_app flask run
```

The application runs locally at:

```text
http://127.0.0.1:5000
```

---

## Running Tests

Run the complete test suite:

```bash
pytest tests/
```

For verbose output:

```bash
pytest tests/ -v
```

Individual test files can also be run during debugging:

```bash
pytest tests/test_streaks.py -v
pytest tests/test_playlists.py -v
```

---

# Bugs Fixed

## 1. Listening Streak Reset on Sundays

### Problem

A user's listening streak worked normally during the week but incorrectly reset when a consecutive listening day occurred on Sunday.

### Root Cause

The streak logic contained an unnecessary weekday condition.

Instead of checking only whether the user listened on the previous day, the condition also required that the current day was **not Sunday**.

This caused legitimate Saturday → Sunday streaks to fall into the reset branch.

### Fix

I removed the Sunday-specific condition so that any consecutive-day listening event correctly increments the streak.

### Verification

I ran:

```bash
pytest tests/test_streaks.py -v
```

All five streak tests passed after the fix, including:

* New streak creation
* Consecutive-day increases
* Same-day listens
* Missed-day resets
* Saturday → Sunday streak progression

---

## 2. Incorrect "Friends Listening Now" Window

### Problem

The "Friends Listening Now" feed showed users who had listened several hours earlier.

Someone who listened two hours ago could still appear as if they were currently listening.

### Root Cause

The service used a recency threshold of:

```python
timedelta(hours=24)
```

A 24-hour window was far too large for a feature intended to represent people listening **now**.

### Fix

I changed the threshold to:

```python
timedelta(minutes=30)
```

### Verification

I manually tested the service using Flask shell with seeded users whose listening times fell on opposite sides of the threshold.

A friend who listened about 15 minutes earlier remained visible, while a friend who listened roughly two hours earlier was correctly removed.

I also verified that the separate activity feed was unaffected.

---

## 3. Missing Rating Notifications

### Problem

Users received notifications when someone added their shared song to a playlist, but they received nothing when someone rated that same song.

### Root Cause

The existing playlist workflow explicitly created a notification for the original song sharer.

The `rate_song()` workflow saved the rating but contained **no equivalent notification step**.

The notification system itself was working; the rating flow simply never called it.

### Fix

I added notification creation to the rating workflow.

After a rating is saved, the application now notifies the person who originally shared the song.

The implementation also prevents users from receiving notifications for rating their own songs.

### Verification

I manually tested the behavior using Flask shell.

I confirmed that:

* Rating another user's song creates a `song_rated` notification.
* The notification goes to the original song sharer.
* Rating your own song does not create a notification.
* Existing rating behavior still works.

---

## 4. Last Playlist Song Missing

### Problem

A playlist containing five songs returned only four.

The missing item was consistently the final song.

### Root Cause

The database query correctly retrieved every song, but the service returned:

```python
songs[:-1]
```

Python's `[:-1]` slice means:

```text
every item except the last one
```

So the application correctly queried the playlist and then accidentally removed its final song immediately before returning the results.

### Fix

I removed the slice and returned the complete song list.

### Verification

I ran:

```bash
pytest tests/test_playlists.py -v
```

All playlist tests passed.

I verified that:

* All songs are returned
* Playlist ordering is preserved
* Empty playlists still return an empty list correctly

---

## Debugging Approach

Rather than changing code based only on the issue descriptions, I traced each feature through the application's architecture:

```text
Route → Service → Model → Database
```

My debugging process generally involved:

1. Reproducing the reported behavior
2. Identifying the relevant route and service
3. Reading the existing implementation
4. Comparing expected and actual behavior
5. Identifying the root cause
6. Making the smallest targeted fix
7. Running tests or manually exercising the service
8. Checking related behavior for regressions

Some bugs were covered by existing pytest tests, while others required manual reproduction through `flask shell`.

---

## Example Feature Flow

One pattern I learned to look for was that user actions often trigger additional behavior behind the scenes.

For example:

```text
POST /songs/<id>/listen
        ↓
routes/songs.py
        ↓
streak_service.record_listening_event()
        ↓
Record listening event
        +
Update listening streak
```

Likewise:

```text
Rate another user's song
        ↓
notification_service.rate_song()
        ↓
Save rating
        +
Create notification for original sharer
```

Understanding these side effects was important because some bugs were not in the obvious action itself, but in secondary behavior triggered by that action.

---

## Detailed Root-Cause Analysis

A more detailed breakdown of each bug, reproduction steps, debugging process, manual verification, and AI usage is available in:

[`submission.md`](./submission.md)

---

## Development Workflow

This project gave me experience with:

* Reading and understanding an unfamiliar Flask codebase
* Tracing requests across routes, services, models, and database operations
* Reproducing bugs before changing code
* Identifying root causes rather than patching symptoms
* Debugging date and time logic
* Fixing conditional logic
* Identifying off-by-one errors
* Implementing missing business behavior
* Testing with pytest
* Manually testing services through Flask shell
* Checking edge cases and regression behavior
* Using feature branches and pull requests
* Writing separate conventional commits for individual fixes

---

## Project Background

Mixtape originated from a starter repository provided through CodePath's AI Engineering curriculum for a debugging and code-maintenance exercise.

The original Flask application, routes, models, services, seed data, and tests were provided as the starting codebase.

My work focused on investigating reported issues, understanding the existing architecture, identifying root causes, implementing four service-layer fixes, and verifying their behavior through automated and manual testing.

The repository remains connected to the original CodePath repository so its source and development history remain transparent.
