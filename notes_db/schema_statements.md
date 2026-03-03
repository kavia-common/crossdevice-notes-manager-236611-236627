# PostgreSQL schema statements (apply one at a time)

These statements are intended to be executed individually, e.g.:

```bash
psql postgresql://... -c "STATEMENT_HERE"
```

---

## Extensions

```sql
CREATE EXTENSION IF NOT EXISTS citext;
```

---

## users

```sql
CREATE TABLE IF NOT EXISTS users (
  id BIGSERIAL PRIMARY KEY,
  email CITEXT NOT NULL,
  password_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```sql
CREATE UNIQUE INDEX IF NOT EXISTS users_email_unique_idx ON users (email);
```

```sql
CREATE INDEX IF NOT EXISTS users_created_at_idx ON users (created_at DESC);
```

---

## notes

```sql
CREATE TABLE IF NOT EXISTS notes (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title TEXT NOT NULL DEFAULT '',
  content TEXT NOT NULL DEFAULT '',
  is_archived BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```sql
CREATE INDEX IF NOT EXISTS notes_user_updated_at_idx ON notes (user_id, updated_at DESC);
```

```sql
CREATE INDEX IF NOT EXISTS notes_user_created_at_idx ON notes (user_id, created_at DESC);
```

```sql
CREATE INDEX IF NOT EXISTS notes_user_archived_updated_at_idx ON notes (user_id, is_archived, updated_at DESC);
```

### Full-text search index (title + content)

```sql
CREATE INDEX IF NOT EXISTS notes_search_gin_idx
ON notes
USING GIN (to_tsvector('english', coalesce(title,'') || ' ' || coalesce(content,'')));
```

---

## tags

```sql
CREATE TABLE IF NOT EXISTS tags (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```sql
CREATE UNIQUE INDEX IF NOT EXISTS tags_user_name_unique_idx ON tags (user_id, lower(name));
```

```sql
CREATE INDEX IF NOT EXISTS tags_user_name_idx ON tags (user_id, lower(name));
```

---

## note_tags (join)

```sql
CREATE TABLE IF NOT EXISTS note_tags (
  note_id BIGINT NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
  tag_id BIGINT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (note_id, tag_id)
);
```

```sql
CREATE INDEX IF NOT EXISTS note_tags_tag_id_idx ON note_tags (tag_id);
```

```sql
CREATE INDEX IF NOT EXISTS note_tags_note_id_idx ON note_tags (note_id);
```

---

## updated_at trigger helper

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

```sql
DROP TRIGGER IF EXISTS users_set_updated_at ON users;
```

```sql
CREATE TRIGGER users_set_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

```sql
DROP TRIGGER IF EXISTS notes_set_updated_at ON notes;
```

```sql
CREATE TRIGGER notes_set_updated_at
BEFORE UPDATE ON notes
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

```sql
DROP TRIGGER IF EXISTS tags_set_updated_at ON tags;
```

```sql
CREATE TRIGGER tags_set_updated_at
BEFORE UPDATE ON tags
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```
