# Database Schema

## Engine

The database engine is SQLite (version 3).  
The database file is stored at `~/.carrysprint/data.db`.  
The database file is created automatically on first server start.  
All migrations run automatically at server startup.

## Encoding

The database uses UTF-8 encoding.  
All datetime values are stored in UTC as ISO 8601 strings.  
All date values are stored as `YYYY-MM-DD` strings.

## Migrations

Migrations are numbered sequentially starting from `0001`.  
Each migration is applied exactly once.  
The applied migrations are tracked in the `schema_migrations` table.

---

## Tables

### schema_migrations

Tracks applied database migrations.

```sql
CREATE TABLE schema_migrations (
    version     TEXT    NOT NULL PRIMARY KEY,
    applied_at  TEXT    NOT NULL
);
```

| Column | Type | Constraints | Description |
|---|---|---|---|
| `version` | TEXT | PRIMARY KEY | Migration version string, e.g. `0001` |
| `applied_at` | TEXT | NOT NULL | UTC datetime when migration was applied |

---

### sprints

Stores sprint records.

```sql
CREATE TABLE sprints (
    id          INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    name        TEXT    NOT NULL,
    start_date  TEXT    NOT NULL,
    end_date    TEXT    NOT NULL,
    status      TEXT    NOT NULL DEFAULT 'planned',
    created_at  TEXT    NOT NULL,
    updated_at  TEXT    NOT NULL
);
```

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | Auto-incremented sprint ID |
| `name` | TEXT | NOT NULL | Sprint name |
| `start_date` | TEXT | NOT NULL | Start date in `YYYY-MM-DD` format |
| `end_date` | TEXT | NOT NULL | End date in `YYYY-MM-DD` format |
| `status` | TEXT | NOT NULL, DEFAULT `'planned'` | Sprint lifecycle status |
| `created_at` | TEXT | NOT NULL | UTC creation datetime |
| `updated_at` | TEXT | NOT NULL | UTC last-update datetime |

**Allowed values for `status`:** `planned`, `active`, `completed`

---

### tasks

Stores task records belonging to a sprint.

```sql
CREATE TABLE tasks (
    id          INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    sprint_id   INTEGER NOT NULL REFERENCES sprints(id) ON DELETE CASCADE,
    title       TEXT    NOT NULL,
    description TEXT    NOT NULL DEFAULT '',
    status      TEXT    NOT NULL DEFAULT 'todo',
    priority    TEXT    NOT NULL DEFAULT 'medium',
    estimate    INTEGER,
    created_at  TEXT    NOT NULL,
    updated_at  TEXT    NOT NULL
);
```

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | Auto-incremented task ID |
| `sprint_id` | INTEGER | NOT NULL, FOREIGN KEY | Owning sprint ID |
| `title` | TEXT | NOT NULL | Task title |
| `description` | TEXT | NOT NULL, DEFAULT `''` | Optional free-text description |
| `status` | TEXT | NOT NULL, DEFAULT `'todo'` | Task lifecycle status |
| `priority` | TEXT | NOT NULL, DEFAULT `'medium'` | Task priority level |
| `estimate` | INTEGER | nullable | Estimated story points |
| `created_at` | TEXT | NOT NULL | UTC creation datetime |
| `updated_at` | TEXT | NOT NULL | UTC last-update datetime |

**Allowed values for `status`:** `todo`, `in_progress`, `done`  
**Allowed values for `priority`:** `low`, `medium`, `high`

---

## Indexes

```sql
CREATE INDEX idx_tasks_sprint_id ON tasks(sprint_id);
CREATE INDEX idx_tasks_status    ON tasks(status);
CREATE INDEX idx_sprints_status  ON sprints(status);
```

## Cascades

Deleting a sprint deletes all its tasks via `ON DELETE CASCADE`.

## Constraints

The `start_date` must be less than or equal to `end_date` (enforced at the application layer).  
The `name` column of `sprints` must not be empty (enforced at the application layer).  
The `title` column of `tasks` must not be empty (enforced at the application layer).
