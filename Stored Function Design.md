---
tags: [dashboard, postgres, design, stored-functions]
---

# Stored function design

> [!info] Purpose
> Five stored functions in the `ops` schema are the entire interface
> between applications, the database, and the dashboard. This note
> explains what each does and — more importantly — *why* a few design
> decisions were made, so the reasoning doesn't have to be
> rediscovered later.

## Write functions

Called by applications to report what's happening. Each call reports a
single fact "as of now" — no history management, no resets, nothing
computed.

### `update_state`

```python
ops.update_state(customer_name, component_type, component_name, available) -> void
```

Writes the current up/down state for one component. Overwrites
whatever was there before — state has no history, just a current
value. See [[State vs Stats Architecture]].

### `update_stats`

```python
ops.update_stats(customer_name, component_type, component_name,
                  total_events, total_errors, total_response_time_ms,
                  [p_interval]) -> void
```

Adds a batch of stats into whichever time window is currently active.
`p_interval` has a default (currently 5 minutes) and callers never
need to pass it.

> [!important] `total_response_time_ms` is a SUM, not an average
> This was the original design's naming mistake — it used to be called
> `average_response_time_ms`. The problem: averages can't be correctly
> accumulated across multiple calls into the same window without also
> tracking each call's sample size and doing a weighted-average
> calculation. Storing the raw **sum** alongside `total_events` keeps
> `update_stats` a pure, zero-cost increment — exactly the property we
> wanted, inherited from the original NonStop shared-memory-counter
> design. The true per-event average is computed once, at read time,
> as `total_response_time_ms / total_events`. See
> [[Stats Windowing and Resets]].

## Read functions

Called by the dashboard to display current state and stats. Each
returns rows ready to render — no further computation should happen
in the Flask app or the template beyond mapping a value to an arrow or
a colour.

### `get_customer_list`

```python
ops.get_customer_list() -> TABLE(customer_id, customer_name)
```

Every customer, for the dashboard's selection dropdown. No filtering.

### `get_customer_state`

```python
ops.get_customer_state(customer_id) -> TABLE(component_id, component_name,
                                              component_type, state, reported_at)
```

Current state, one row per component. Returns `state` exactly as
`update_state` last wrote it (`'UP'` / `'DOWN'`).

### `get_customer_stats`

```python
ops.get_customer_stats(customer_id) -> TABLE(component_id, component_name,
                                              component_type, window_start,
                                              sample_count, total_events,
                                              total_errors, total_response_time_ms,
                                              avg_response_time_ms, success_rate,
                                              reported_at)
```

Most recent stats window, one row per component. Computes two derived
values before returning:
- `avg_response_time_ms` = `total_response_time_ms / total_events`
- `success_rate` = `100 - (total_errors / total_events) * 100`

> [!important] Why these joins are INNER, not LEFT
> Both read functions join from `ops.component` to `ops.state` /
> `ops.stats` using an **inner** join. A component only appears in the
> State section if it has an actual state row, and only appears in the
> Stats section if it has an actual stats row.
>
> This came from a real bug: with a `LEFT JOIN`, *every* component
> showed up in *both* sections — a stats-only channel appeared in
> State as a meaningless grey "unknown," and a state-only orchestrator
> appeared in Stats as a row of dashes. Switching to `INNER JOIN` made
> "no row" mean exactly what it should: *this component doesn't
> participate in that dimension*, not *something's missing*.
>
> Trade-off worth knowing: a component that's genuinely new and simply
> hasn't reported yet will also be silently absent, rather than shown
> as "unknown." That's an acceptable gap for now — revisit if it
> becomes a real problem once components can be added ahead of their
> first report.

## What's deliberately NOT in these functions

- **No thresholds or colour logic.** `success_rate` is a plain
  arithmetic derivation (division and subtraction) computed here
  because it's just arithmetic on stored facts — but deciding that
  95%+ is "green" and below 75% is "red" happens in the dashboard
  template, not here. Arithmetic derivation lives in the data layer;
  judgment calls live in the presentation layer. See the visualization
  architecture discussion in [[Welcome]].
- **No auto-creation of customers or components.** Calling
  `update_stats`/`update_state` for an unknown customer or component
  raises an error rather than silently inserting one. Reference data
  is managed deliberately, not implicitly through the reporting API.

## A pitfall worth knowing before changing any of these

> [!danger] `CREATE OR REPLACE FUNCTION` only replaces an exact signature match
> Postgres identifies a function by name **and** argument types
> together. Changing a parameter's type (e.g. `bigint` → `integer`) or
> adding/removing a parameter means `CREATE OR REPLACE` silently
> creates a *second* overload instead of replacing the first — both
> then exist, and Postgres picks whichever it considers the best match
> for a given call, which is rarely what you expect.
>
> Before changing any function's parameter list, check what's
> currently deployed:
> ```sql
> SELECT oid::regprocedure FROM pg_proc
> WHERE pronamespace = 'ops'::regnamespace AND proname = 'update_stats';
> ```
> If more than one row comes back, drop the stale one explicitly
> (`DROP FUNCTION ops.update_stats(<exact old signature>);`) before
> deploying the new version.

## See also

- [[Creating the Database Schema]]
- [[Stats Windowing and Resets]]
- [[State vs Stats Architecture]]
- [[Using ops_stats from Python]]
