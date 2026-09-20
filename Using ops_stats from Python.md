---
tags: [dashboard, python, api, ops_stats]
---

# Using `ops_stats` from a Python application

> [!info] Purpose
> This note explains how an application reports **state** (up/down) and
> **statistics** (events, errors, response time) into the dashboard's
> Postgres backend, using the `ops_stats` Python package.

There are exactly two entry points. Everything else — windowing, resets,
averages, success rate — happens server-side in stored functions. The
application never computes any of that; it just reports raw facts.

```python
from ops_stats import (
    update_stats,
    update_state,
)
```

## Prerequisites

> [!warning] Before calling either function
> - The `ops_stats` package is installed in the application's Python
>   environment (`pip install ops_stats`, or whatever internal index
>   hosts it).
> - `ops.customer` already has a row for the customer you're about to
>   report for (e.g. `BlueFez`, `Platform42`). Reporting for an unknown
>   customer name raises an error rather than silently creating one —
>   see [[Stored Function Design#update_stats]].
> - `ops.component` already has a row for that customer + component
>   type + component name combination (e.g. `CHANNEL` / `WhatsApp`,
>   `ORCHESTRATOR` / `Orchestrator`). Same rule: unknown components
>   raise, they aren't auto-created.
> - Connection details (`PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`,
>   `PGPASSWORD`) are available via environment variables or a `.env`
>   file. See [[Connecting to the Database]].

## Reporting statistics: `update_stats`

```python
update_stats(
    customer_name="BlueFez",
    component_type="CHANNEL",
    component_name="WhatsApp",
    total_events=1,
    total_errors=1,
    total_response_time_ms=100.0,
)
```

| Parameter | Meaning |
|---|---|
| `customer_name` | Must match an existing row in `ops.customer`. |
| `component_type` | e.g. `"CHANNEL"`, `"ORCHESTRATOR"`. Must match `ops.component`. |
| `component_name` | e.g. `"WhatsApp"`, `"Instagram"`, `"Orchestrator"`. Must match `ops.component`. |
| `total_events` | Number of items processed **in this batch** — not a running total. |
| `total_errors` | Number of those that errored, in the same batch. |
| `total_response_time_ms` | **Sum** of response times across the batch — not an average. |

> [!important] `total_response_time_ms` is a sum, not an average
> If an app batches 100–1000 events before reporting (the "event
> hysteresis" pattern), it sums the response times of everything in
> that batch and passes the sum. The stored function accumulates sums
> and counts separately, and the dashboard derives the true
> per-event average as `total_response_time_ms / total_events` at
> read time. See [[Stats Windowing and Resets]] for why averages can't
> be accumulated directly.

Each call is a **batch report**, not a snapshot — `update_stats` adds
these numbers into whatever time window is currently active
server-side (5 minutes by default). You call it every time you have a
batch to report; you never need to reset, zero out, or manage windows
yourself.

## Reporting state: `update_state`

```python
update_state(
    customer_name="BlueFez",
    component_type="ORCHESTRATOR",
    component_name="Orchestrator",
    available=False,
)
```

| Parameter | Meaning |
|---|---|
| `customer_name` | Same matching rule as `update_stats`. |
| `component_type` / `component_name` | Same matching rule as `update_stats`. |
| `available` | `True` = up, `False` = down. Stored server-side as `'UP'` / `'DOWN'`. |

Unlike stats, state is **not windowed** — there's no history of past
states, just the current value, overwritten on every call. See
[[State vs Stats Architecture]] for why these two are modeled so
differently.

## Full example (matches the two demo customers)

```python
from ops_stats import update_stats, update_state

# BlueFez: one WhatsApp message processed, with an error, 100ms response
update_stats(
    customer_name="BlueFez",
    component_type="CHANNEL",
    component_name="WhatsApp",
    total_events=1,
    total_errors=1,
    total_response_time_ms=100.0,
)

# BlueFez: orchestrator is down
update_state(
    customer_name="BlueFez",
    component_type="ORCHESTRATOR",
    component_name="Orchestrator",
    available=False,
)

# Platform42: WhatsApp batch of 3, 1 error, summed 180ms
update_stats(
    customer_name="Platform42",
    component_type="CHANNEL",
    component_name="WhatsApp",
    total_events=3,
    total_errors=1,
    total_response_time_ms=180.0,
)

# Platform42: Instagram batch of 4, no errors, summed 55ms
update_stats(
    customer_name="Platform42",
    component_type="CHANNEL",
    component_name="Instagram",
    total_events=4,
    total_errors=0,
    total_response_time_ms=55.0,
)

# Platform42: orchestrator is up
update_state(
    customer_name="Platform42",
    component_type="ORCHESTRATOR",
    component_name="Orchestrator",
    available=True,
)
```

## One-shot calls vs. a reusable connection

Both `update_stats` and `update_state` shown above are **one-shot
convenience functions** — each call opens a connection, does its work,
commits, and closes. That's the right choice for occasional calls (a
script, a webhook handler).

For a long-running process reporting repeatedly (a daemon, a service
loop), reuse a single connection instead with `OpsClient`:

```python
from ops_stats import OpsClient

with OpsClient() as client:
    client.update_stats(
        customer_name="BlueFez",
        component_type="CHANNEL",
        component_name="WhatsApp",
        total_events=1,
        total_errors=1,
        total_response_time_ms=100.0,
    )
    client.update_state(
        customer_name="BlueFez",
        component_type="ORCHESTRATOR",
        component_name="Orchestrator",
        available=False,
    )
```

## What the application never has to think about

- Which time window a stats report lands in — the stored function
  works that out from `now()`.
- Resetting counters — there's no reset call; a new window simply
  starts a fresh row.
- Computing averages or success rates — those are derived at
  dashboard-read time, not at report time.
- Whether a component "only" reports state or "only" reports stats —
  it just calls whichever function applies; the dashboard's read side
  handles components that don't participate in one dimension. See
  [[State vs Stats Architecture]].

## See also

- [[Stored Function Design]]
- [[Stats Windowing and Resets]]
- [[State vs Stats Architecture]]
- [[Dashboard Architecture Overview]]
