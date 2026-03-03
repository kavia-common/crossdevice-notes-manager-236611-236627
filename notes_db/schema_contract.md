# Notes DB (PostgreSQL) — Schema & Connection Contract

This container runs PostgreSQL for the cross-device notes app.

## Connection contract (for backend)

The backend **must** connect using the values exposed by this DB container:

- `POSTGRES_URL`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `POSTGRES_DB`
- `POSTGRES_PORT`

A ready-to-use `psql` connection command is also written to:

- `notes_db/db_connection.txt`

Example format:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp
```

### Expected host/port
Within this repository's dev environment, PostgreSQL is configured to listen on:

- Host: `localhost`
- Port: `$POSTGRES_PORT` (currently `5000` in startup.sh)

## Schema overview

Tables:

- `users` — application users (email/password hash)
- `notes` — notes authored by a user (supports autosave via `updated_at`)
- `tags` — per-user tags
- `note_tags` — many-to-many join between notes and tags

All tables include `created_at` and `updated_at` timestamps.

### Key constraints & behavior

- Emails are unique (case-insensitive via `citext`).
- Tags are unique **per user** (same tag name allowed for different users).
- Deleting a user deletes their notes/tags and related join rows.
- Deleting a note deletes its note_tags rows.
- Deleting a tag deletes its note_tags rows.
- Notes enforce `updated_at` refresh on UPDATE via a trigger.

## Applying the schema

This repository follows a simple, explicit approach:
- Apply DDL **one statement at a time** using `psql -c`.

The canonical DDL statements are captured in `notes_db/schema_statements.md`.

Typical flow:

```bash
cd notes_db
source db_visualizer/postgres.env 2>/dev/null || true
$(cat db_connection.txt) -c "SELECT 1;"
# Then apply statements from schema_statements.md one by one
```

## Indexing notes search

This schema includes:
- a `GIN` index for full-text search on note title + content (English config)
- supporting btree indexes for common filters/sorts:
  - notes by (user_id, updated_at desc)
  - tags by (user_id, name)
  - note_tags by (tag_id) and (note_id)

If you later add advanced search requirements (prefix search, multiple languages, ranking),
consider a dedicated `tsvector` column with generated expressions.
For now, an expression index is used to keep the schema minimal.
