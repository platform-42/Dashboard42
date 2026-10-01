---
tags: [dashboard, home, overview]
---

# Dashboard — Welcome

> [!info] What this is
> A lightweight, Geckoboard-style status dashboard for monitoring
> customer components (channels like WhatsApp/Instagram, the
> orchestrator, etc.). Green up-arrow = good, red down-arrow = bad,
> dark gray = planned maintenance (guarded, but no action needed).
> Built as a proof of concept: no authentication yet, one customer
> viewed at a time, selected from a dropdown.

## How it fits together

```
Application (reports data)
        │  ops_stats Python package
        ▼
Postgres: ops schema
  - ops.update_stats() / ops.update_state()   <- write side
  - ops.get_customer_state() / ops.get_customer_stats()  <- read side
        │
        ▼
Flask dashboard (Bootstrap cards, 5-minute auto-refresh)
  runs under gunicorn on 127.0.0.1:8000 (not reachable from outside)
        ▲
        │  proxy_pass  <- routing configured in the nginx config
        │
nginx (reverse proxy: HTTPS, slow clients, access control)
        ▲
        │  HTTPS
        │
Browser (workstation / iPhone)
```

Visitors never talk to Flask directly: nginx is the only public entry
point and forwards each request to gunicorn. See
[[Nginx Reverse Proxy]] for why, and how the routing is set up.

Two independent things are tracked per component:

- **State** — current up/down, not windowed, always "right now."
- **Stats** — events/errors/response time, accumulated into rolling
  time windows, reset automatically by the passage of time (not an
  explicit reset call).

See [[State vs Stats Architecture]] for why these are modeled so
differently.

## Setup — six steps

> [!todo] Follow these in order on a fresh machine.

### 1. Install PostgreSQL

Get a Postgres server running locally (or point at an existing one).
→ [[Installing PostgreSQL]]

### 2. Create the database, schema, tables, and stored functions

Sets up the `ops` schema and its five functions:
- `ops.update_stats()` — write a batch of stats
- `ops.update_state()` — write current up/down state
- `ops.get_customer_list()` — list customers, for the dashboard's picker
- `ops.get_customer_state()` — read current state per component
- `ops.get_customer_stats()` — read latest stats window per component

→ [[Creating the Database Schema]]

### 3. Install the dashboard software (the Flask service)

The lightweight web app that renders the cards and auto-refreshes.
→ [[Installing the Dashboard]]

Needs the same database connection details as step 4 — see
[[Database Environment Variables]].

### 4. Install the dashboard API (`ops_stats`)

The Python package that *any* application uses to report state and
stats into the database.
→ [[Installing ops_stats]]
→ [[Using ops_stats from Python]]

### 5. Verify it end to end

Report some test data, then open the dashboard and confirm it shows
up:

```python
from ops_stats import update_stats, update_state

update_stats(
    customer_name="BlueFez",
    component_type="CHANNEL",
    component_name="WhatsApp",
    total_events=1,
    total_errors=0,
    total_response_time_ms=42.0,
)
update_state(
    customer_name="BlueFez",
    component_type="ORCHESTRATOR",
    component_name="Orchestrator",
    available=True,
)
```

Then run the Flask app and open it in a browser — pick **BlueFez**
from the dropdown and confirm the WhatsApp stats card and the
Orchestrator state card both appear.

### 6. Put nginx in front (production)

Run the dashboard under gunicorn and let nginx handle HTTPS and the
public traffic. Locally on the Mac, open `http://localhost:8080`.
→ [[Nginx Reverse Proxy]]

## Reference

- [[Stored Function Design]]
- [[Stats Windowing and Resets]]
- [[State vs Stats Architecture]]
- [[Database Environment Variables]]
- [[Installing ops_stats]]
- [[Using ops_stats from Python]]
- [[Nginx Reverse Proxy]]
