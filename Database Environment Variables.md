---
tags: [dashboard, python, postgres, config, security]
---

# Database environment variables

> [!info] Purpose
> Both the Flask dashboard and any application calling `ops_stats`
> connect to Postgres via `psycopg`, which reads connection details
> from standard libpq environment variables. These need to be set
> *before* the application runs.

## The variables

```bash
export PGHOST=localhost
export PGPORT=5432
export PGDATABASE=dashboard
export PGUSER=postgres
export PGPASSWORD='CHANGE_ME_STRONG_PASSWORD'
```

| Variable | Meaning |
|---|---|
| `PGHOST` | Database server hostname. |
| `PGPORT` | Database server port (Postgres default: `5432`). |
| `PGDATABASE` | Database name. |
| `PGUSER` | Database role to connect as. |
| `PGPASSWORD` | Password for that role. |

## Two common ways to load them

**A) Shell script, sourced manually** — the form above, saved as e.g.
`env.sh` and loaded with:

```bash
source env.sh
```

**B) `.env` file**, read automatically by `python-dotenv` (used by
`ops_stats` and the dashboard app via `load_dotenv()`). Same values,
no `export` needed:

```
PGHOST=localhost
PGPORT=5432
PGDATABASE=dashboard
PGUSER=postgres
PGPASSWORD=CHANGE_ME_STRONG_PASSWORD
```

Drop this file as `.env` next to the script or app — it's picked up
automatically, no code change needed. See
[[Using ops_stats from Python]].

## Local development vs. production

> [!warning] A plaintext `.env` (or `env.sh`) file is fine for local
> development only.
> - Never commit it — add `.env` and `env.sh` to `.gitignore`.
> - `chmod 600` the file so only your own user account can read it.
>
> Neither of those is sufficient for production. In production, **the
> file itself or the password inside it must be encrypted at rest**,
> not just access-restricted. Options, roughly in order of effort:
> - A secrets manager (AWS Secrets Manager, HashiCorp Vault, GCP
>   Secret Manager) that injects `PGPASSWORD` into the process
>   environment at startup — nothing sensitive ever touches disk.
> - An encrypted secrets file (e.g. `sops`, `age`) that's decrypted
>   only at deploy time.
> - At an absolute minimum, if a plaintext file must exist on a
>   production host, it should sit on an encrypted volume with strict
>   file permissions — this is the weakest option and only a stopgap.

> [!danger] Rotate any credential that's ever been written down in a
> shared place — a doc, a vault note, a chat log, a ticket — even if
> access to that place is restricted. Treat it as compromised and
> issue a new one.

## See also

- [[Installing ops_stats]]
- [[Using ops_stats from Python]]
