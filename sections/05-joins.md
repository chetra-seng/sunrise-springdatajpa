---
layout: center
---
# Primary Keys

## Uniquely Identifying Every Row

---

# What is a Primary Key?

Every row needs a **unique identifier** so you can reference it precisely.

```sql
CREATE TABLE tasks (
    id          BIGSERIAL    PRIMARY KEY,
    title       VARCHAR(255) NOT NULL,
    ...
);
```

- `PRIMARY KEY` → unique across the table, never `NULL`
- `BIGSERIAL` → PostgreSQL auto-assigns the next integer (1, 2, 3, …)
- You never INSERT a value for `id` — the database handles it

---

# BIGSERIAL — Auto-Increment

```sql
-- BIGSERIAL is shorthand for:
id BIGINT NOT NULL DEFAULT nextval('tasks_id_seq') PRIMARY KEY
```

PostgreSQL maintains a **sequence** — a counter that increments with every insert.

```sql
INSERT INTO tasks (title, completed) VALUES ('Task A', false);  -- gets id = 1
INSERT INTO tasks (title, completed) VALUES ('Task B', false);  -- gets id = 2
INSERT INTO tasks (title, completed) VALUES ('Task C', false);  -- gets id = 3
```

IDs are never reused — even if you delete a row, its ID is gone.

---

# What Happens When You INSERT

```sql
INSERT INTO tasks (title, description, completed)
VALUES ('Learn SQL', 'Practice queries', false);
```

PostgreSQL:
1. Increments the sequence → next value is, say, `4`
2. Inserts the row with `id = 4`, `created_at = NOW()`
3. Returns the new row

You can retrieve it immediately:

```sql
INSERT INTO tasks (title, completed) VALUES ('New task', false)
RETURNING id, created_at;
```

---

# Composite Primary Keys

Sometimes a primary key spans two columns:

```sql
CREATE TABLE task_tags (
    task_id  BIGINT REFERENCES tasks(id),
    tag_id   BIGINT REFERENCES tags(id),
    PRIMARY KEY (task_id, tag_id)   -- the pair must be unique
);
```

No separate `id` column — the combination of `task_id + tag_id` is the identity.

---

# Practice

Given:
```sql
CREATE TABLE projects (
    id          BIGSERIAL    PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    created_at  TIMESTAMP    NOT NULL DEFAULT NOW()
);
```

1. Write an INSERT that creates a project named "Sunrise Course"
2. What `id` will the first project get?
3. What happens if you run the INSERT a second time with the same name?
4. Can two projects ever have the same `id`?

<v-click>

```sql
-- 1.
INSERT INTO projects (name) VALUES ('Sunrise Course');
-- 2. id = 1  (first row, sequence starts at 1)
-- 3. A second row is inserted — no uniqueness constraint on name, id = 2
-- 4. No — PRIMARY KEY enforces uniqueness
```

</v-click>
