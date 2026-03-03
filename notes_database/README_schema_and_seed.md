# Notes Database (PostgreSQL) — Schema + Demo Seed

This container uses a local PostgreSQL instance started by `startup.sh` and exposes connection info via `db_connection.txt`.

## Connection convention (IMPORTANT)

`db_connection.txt` contains a convenient `psql ...` command, e.g.:

```
psql postgresql://appuser:dbuser123@localhost:5000/myapp
```

When scripting, you typically want the URI without the leading `psql `:

```bash
CONN_RAW=$(cat db_connection.txt)
CONN=${CONN_RAW#psql }
PGPASSWORD="dbuser123" psql "$CONN" -c "SELECT 1;"
```

## Extensions

Schema uses UUID primary keys via `gen_random_uuid()`, so ensure:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

## Schema (tables)

### users
- One row per application user.
- `email` is unique (login identifier).

Columns:
- `id uuid PK default gen_random_uuid()`
- `email text unique not null`
- `password_hash text not null` (bcrypt/argon2 hash stored by backend)
- `display_name text null`
- `created_at/updated_at timestamptz`

### notes
- Owned by a user (`user_id`), cascade deletes.
- Full-text search supported via generated `search_vector`.

Columns:
- `id uuid PK default gen_random_uuid()`
- `user_id uuid FK -> users(id) on delete cascade`
- `title text not null`
- `content text not null`
- `content_format text not null default 'markdown'`
- `is_archived boolean not null default false`
- `created_at/updated_at timestamptz`
- `search_vector tsvector GENERATED ALWAYS AS (...) STORED`

### tags
- Owned by a user (`user_id`), cascade deletes.
- Tag names are unique *per user*.

Columns:
- `id uuid PK default gen_random_uuid()`
- `user_id uuid FK -> users(id) on delete cascade`
- `name text not null`
- `color text null`
- `created_at timestamptz`
- `UNIQUE(user_id, name)`

### note_tags (many-to-many)
- Join table connecting notes and tags.
- Cascade deletes from both sides.

Columns:
- `note_id uuid FK -> notes(id) on delete cascade`
- `tag_id uuid FK -> tags(id) on delete cascade`
- `created_at timestamptz`
- `PRIMARY KEY(note_id, tag_id)`

## Indexes (performance)

- Notes listing per user (recently updated):
  - `notes(user_id, updated_at DESC)`
- Full-text search:
  - `GIN(notes.search_vector)`
- Tag list/lookup per user:
  - `tags(user_id, name)`
- Query notes by tag:
  - `note_tags(tag_id)`

## Demo seed data

Minimal seed is intended for local/demo use and is **idempotent** via `ON CONFLICT`:

- Demo user: `demo@notes.local`
- Tags: `work`, `personal`
- Notes: "Welcome to Notes", "Work Ideas"
- note_tags associations to demonstrate tag filtering

The demo user's `password_hash` is just a placeholder bcrypt-like string; the backend should overwrite/store real hashes on registration.

## How to (re)apply manually

Per the project guidance, run SQL statements **one at a time** via `psql -c`.

If you need to recreate the schema from scratch, you can drop tables in dependency order:

```sql
DROP TABLE IF EXISTS note_tags;
DROP TABLE IF EXISTS tags;
DROP TABLE IF EXISTS notes;
DROP TABLE IF EXISTS users;
```

Then re-run the CREATE TABLE / CREATE INDEX statements and seed inserts.
