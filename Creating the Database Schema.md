---
tags: [dashboard, postgres, database, schema, git]
---

# Creating the database schema

> [!info] Source
> The database, schema, tables, and stored functions are all defined
> as SQL files in a dedicated git repository:
> `https://github.com/platform-42/dashboard_dbms.git`


## Clone the repo

```bash
git clone https://github.com/platform-42/dashboard_dbms.git
cd dashboard_dbms
```

## Create the database

Before running any DDL, the database itself needs to exist:

```bash
createdb dashboard
```

> [!note] The repo's own README also mentions creating a `~/.pgpass`
> file as its first step. We've standardized on environment variables
> instead (see [[Database Environment Variables]]) for consistency
> with how the dashboard app and `ops_stats` are configured — `.pgpass`
> works too, but mixing both in the same setup is what causes
> inconsistent `psql` behaviour. Pick one.

## Run the SQL files, in order

Confirmed from the repo's own `README.md`:

```
\i ddl/dashboard/ops/010-schema.sql
\i ddl/dashboard/ops/020-customer.sql
\i ddl/dashboard/ops/030-component.sql
\i ddl/dashboard/ops/040-state.sql
\i ddl/dashboard/ops/050-stats.sql
```

| # | File | Creates |
|---|---|---|
| 0 | *(n/a — `createdb` above)* | The database itself |
| 1 | `010-schema.sql` | The `ops` schema |
| 2 | `020-customer.sql` | `ops.customer` table |
| 3 | `030-component.sql` | `ops.component` table |
| 4 | `040-state.sql` | `ops.state` table |
| 5 | `050-stats.sql` | `ops.stats` table |

Then all five stored functions, from the `functions/` subfolder, in
numeric order:

```
\i ddl/dashboard/ops/functions/010-update-state.sql
\i ddl/dashboard/ops/functions/020-update-stats.sql
\i ddl/dashboard/ops/functions/030-get-customer-list.sql
\i ddl/dashboard/ops/functions/040-get-customer-stats.sql
\i ddl/dashboard/ops/functions/050-get-customer-state.sql
```

Both the write side (`update_state`, `update_stats` — what an
application calls to report data) and the read side
(`get_customer_list`, `get_customer_state`, `get_customer_stats` —
what the dashboard calls to display it) live together in this one
repo. See [[Installing the Dashboard]] for how the dashboard app
consumes the read-side functions.

## What gets created

Once both DDL sections above have run successfully, five stored
functions:

| Function | Purpose |
|---|---|
| `ops.update_state(...)` | Write current up/down state for a component |
| `ops.update_stats(...)` | Write a batch of stats for a component |
| `ops.get_customer_list()` | List all customers (dashboard's picker) |
| `ops.get_customer_state(customer_id)` | Read current state per component |
| `ops.get_customer_stats(customer_id)` | Read latest stats window per component |

Plus the underlying tables: `ops.customer`, `ops.component`,
`ops.state`, `ops.stats`.

## Verify

```sql
\dn ops                 -- confirm the ops schema exists
\dt ops.*                -- confirm the tables exist
\df+ ops.update_state     -- confirm each function exists, single overload
\df+ ops.update_stats
\df+ ops.get_customer_list
\df+ ops.get_customer_state
\df+ ops.get_customer_stats
```

Each `\df+` should return exactly **one row** per function name. If a
function shows more than one row (multiple overloads), see
[[Stored Function Design]] for how to clean that up.

## Populate reference data

Before anything can be reported, `ops.customer` and `ops.component`
need rows for the customers and components you actually want to
track (matching whatever names your application passes to
`update_stats()` / `update_state()`) — see
[[Using ops_stats from Python]] for why unknown customers/components
raise an error rather than being auto-created.

## See also

- [[Welcome]]
- [[Stored Function Design]]
- [[Database Environment Variables]]
- [[Installing the Dashboard]]
