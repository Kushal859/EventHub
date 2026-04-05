# EventHub — Backend API for Event Ticketing

Why I built this
EventHub is a backend REST API built to understand how a production-grade ticketing system works under the hood. The core engineering problem it solves is seat reservation management — specifically handling the scenario where two users might try to book the last available seat at the same time.
Built using:

Django — web framework, ORM, migrations
Django REST Framework (DRF) — serializers, ViewSets, routers
SQLite — local database
Python 3.11


# How to run the project
Step 1 — clone the repo
bashgit clone https://github.com/Kushal859/EventHub.git
cd EventHub/eventhub
Step 2 — create and activate virtual environment
bash# Windows PowerShell
python -m venv .venv
.venv\Scripts\Activate.ps1

# Mac / Linux
python -m venv .venv
source .venv/bin/activate
Step 3 — install dependencies
bashpip install -r requirements.txt
Step 4 — run migrations
bashpython manage.py migrate
Step 5 — start the server
bashpython manage.py runserver
API is live at http://127.0.0.1:8000/api/

# Project structure
eventhub/
├── eventhub/                 ← Django config package
│   ├── settings.py           ← INSTALLED_APPS, MIDDLEWARE, database
│   ├── urls.py               ← top-level URL entry point
│   ├── wsgi.py
│   └── asgi.py
├── events/                   ← core app
│   ├── models.py             ← Event and Reservation models
│   ├── serializers.py        ← validation + business logic
│   ├── views.py              ← EventViewSet, ReservationViewSet
│   ├── urls.py               ← DefaultRouter wiring
│   ├── middleware.py         ← RequestLoggingMiddleware
│   ├── apps.py               ← app config
│   └── migrations/
│       └── 0001_initial.py   ← auto-generated DB schema
├── manage.py
├── db.sqlite3                ← local SQLite DB (not committed)
├── requirements.txt
└── .gitignore

#All API endpoints
Event endpoints
MethodURLDescriptionGET/api/events/List all eventsPOST/api/events/Create a new eventGET/api/events/{id}/Get one event by IDPUT/api/events/{id}/Fully update an eventPATCH/api/events/{id}/Partially update an eventDELETE/api/events/{id}/Delete an eventGET/api/events/?status=upcomingFilter events by statusGET/api/events/?venue=mumbaiFilter events by venue
# Reservation endpoints
MethodURLDescriptionGET/api/reservations/List all reservationsPOST/api/reservations/Create a reservationGET/api/reservations/{id}/Get one reservation by IDPOST/api/reservations/{id}/cancel/Cancel a reservationGET/api/reservations/?event_id=1Filter reservations by event

# End-to-end test checklist
Run every test below using Postman. Each one must pass before submission.

TEST 1 — Create an event
Request
Method:  POST
URL:     http://127.0.0.1:8000/api/events/
Headers: Content-Type: application/json
Body:
{
    "title": "TechFest 2026",
    "venue": "IIT Bombay, Mumbai",
    "date": "2026-04-05",
    "total_seats": 500,
    "available_seats": 500,
    "status": "upcoming"
}
Expected response: 201 Created
json{
    "id": 1,
    "title": "TechFest 2026",
    "venue": "IIT Bombay, Mumbai",
    "date": "2026-04-05",
    "total_seats": 500,
    "available_seats": 500,
    "status": "upcoming",
    "created_at": "2026-04-05T...",
    "reservations_count": 0
}
What is being tested:

Event model saves correctly to the database
EventSerializer computed field reservations_count returns 0 for a new event
available_seats validation passes when it equals total_seats


TEST 2 — List all events
Request
Method:  GET
URL:     http://127.0.0.1:8000/api/events/
Expected response: 200 OK
json[
    {
        "id": 1,
        "title": "TechFest 2026",
        "reservations_count": 0,
        ...
    }
]
What is being tested:

get_queryset() returns all events when no filters are applied


TEST 3 — Filter events by status
Request
Method:  GET
URL:     http://127.0.0.1:8000/api/events/?status=upcoming
Expected response: 200 OK

Returns only events where status = upcoming
Events with other statuses are excluded

What is being tested:

get_queryset() status filter works correctly


TEST 4 — Filter events by venue
Request
Method:  GET
URL:     http://127.0.0.1:8000/api/events/?venue=mumbai
Expected response: 200 OK

Returns events where venue contains "mumbai" (case-insensitive)

What is being tested:

venue__icontains filter works — partial and case-insensitive match


TEST 5 — Create a reservation (REQUIRED SCREENSHOT)
Request
Method:  POST
URL:     http://127.0.0.1:8000/api/reservations/
Headers: Content-Type: application/json
Body:
{
    "event": 1,
    "attendee_name": "Atharva Khamkar",
    "attendee_email": "atharva@example.com",
    "seats_reserved": 8
}
Expected response: 201 Created
json{
    "id": 1,
    "event": 1,
    "attendee_name": "Atharva Khamkar",
    "attendee_email": "atharva@example.com",
    "seats_reserved": 8,
    "status": "confirmed",
    "created_at": "2026-04-05T..."
}
What is being tested:

validate_seats_reserved passes for value >= 1
validate() confirms event status is upcoming
validate() confirms seats_reserved <= available_seats
create() deducts 8 from event.available_seats (500 → 492)
status is auto-set to confirmed — client cannot override
created_at is auto-set by Django — client cannot override


TEST 6 — Verify seats were deducted
Request
Method:  GET
URL:     http://127.0.0.1:8000/api/events/1/
Expected response: 200 OK
json{
    "id": 1,
    "available_seats": 492,
    "reservations_count": 1,
    ...
}
What is being tested:

available_seats decreased from 500 to 492 after reservation
reservations_count is now 1


TEST 7 — Overbooking attempt (REQUIRED SCREENSHOT)
Request
Method:  POST
URL:     http://127.0.0.1:8000/api/reservations/
Headers: Content-Type: application/json
Body:
{
    "event": 1,
    "attendee_name": "Test User",
    "attendee_email": "test@example.com",
    "seats_reserved": 9999
}
Expected response: 400 Bad Request
json{
    "non_field_errors": [
        "Only 492 seat(s) available."
    ]
}
What is being tested:

Cross-field validate() in ReservationSerializer correctly blocks the request
No seats are deducted from the event
No reservation row is created in the database


TEST 8 — Reserve seats with invalid count
Request
Method:  POST
URL:     http://127.0.0.1:8000/api/reservations/
Headers: Content-Type: application/json
Body:
{
    "event": 1,
    "attendee_name": "Test User",
    "attendee_email": "test@example.com",
    "seats_reserved": 0
}
Expected response: 400 Bad Request
json{
    "seats_reserved": [
        "Must reserve at least 1 seat."
    ]
}
What is being tested:

validate_seats_reserved() field-level validation blocks values less than 1


TEST 9 — Reserve seats for a cancelled event
First update the event status to cancelled:
Method:  PATCH
URL:     http://127.0.0.1:8000/api/events/1/
Body:    { "status": "cancelled" }
Then try to reserve:
Method:  POST
URL:     http://127.0.0.1:8000/api/reservations/
Body:
{
    "event": 1,
    "attendee_name": "Test User",
    "attendee_email": "test@example.com",
    "seats_reserved": 2
}
Expected response: 400 Bad Request
json{
    "non_field_errors": [
        "Cannot reserve seats for a cancelled event."
    ]
}
What is being tested:

validate() blocks reservations for non-active events
Reset event status back to upcoming after this test


TEST 10 — Cancel a reservation (REQUIRED SCREENSHOT)
Request
Method:  POST
URL:     http://127.0.0.1:8000/api/reservations/1/cancel/
No body needed.
Expected response: 200 OK
json{
    "id": 1,
    "event": 1,
    "attendee_name": "Atharva Khamkar",
    "attendee_email": "atharva@example.com",
    "seats_reserved": 8,
    "status": "cancelled",
    "created_at": "2026-04-05T..."
}
What is being tested:

@action(detail=True) custom endpoint works at /reservations/1/cancel/
reservation.status changes from confirmed to cancelled
event.available_seats is restored (492 → 500)


TEST 11 — Cancel an already cancelled reservation
Request
Method:  POST
URL:     http://127.0.0.1:8000/api/reservations/1/cancel/
Expected response: 400 Bad Request
json{
    "error": "Already cancelled."
}
What is being tested:

Guard clause in cancel() prevents double-cancellation
Seats are not double-restored to the event


TEST 12 — Filter reservations by event
Request
Method:  GET
URL:     http://127.0.0.1:8000/api/reservations/?event_id=1
Expected response: 200 OK

Returns only reservations belonging to event with id 1

What is being tested:

get_queryset() event_id filter works correctly


TEST 13 — Middleware logs appear in terminal
After running any of the above requests, check your VS Code terminal. You should see log lines like:
POST /api/events/ - 201 - 0.02s
GET /api/events/ - 200 - 0.01s
POST /api/reservations/ - 201 - 0.04s
POST /api/reservations/1/cancel/ - 200 - 0.03s
What is being tested:

RequestLoggingMiddleware.__call__() runs for every request
Logs show method, path, status code, and response duration in seconds


# Design decision — why seat deduction is in the serializer
The brief notes that in production, transaction.atomic() would wrap both writes to prevent race conditions. For this assignment, event.available_seats -= seats_reserved and Reservation.objects.create() are placed inside ReservationSerializer.create(). This keeps both writes in one method so they always travel together — you cannot accidentally create a reservation without deducting seats.

# Dependencies
django>=4.2
djangorestframework>=3.14
