---
layout: center
---
# CRUD in SQL

## SELECT · INSERT · UPDATE · DELETE

---

# The Four Operations

Every database interaction is one of four things:

| SQL | What It Does |
|-----|-------------|
| `SELECT` | Read data |
| `INSERT` | Add a new row |
| `UPDATE` | Modify an existing row |
| `DELETE` | Remove a row |

---

# Sample Data

```sql
INSERT INTO tasks (title, description, completed) VALUES
  ('Learn Spring Boot',    'Complete the course',  false),
  ('Build TaskController', 'All 7 endpoints',      true),
  ('Write tests',          'MockMvc integration',  false),
  ('Add JPA',              'Migrate from HashMap', false),
  ('Connect PostgreSQL',   'Replace H2 in dev',    true);
```

```
 id │ title                │ completed │ created_at
────┼──────────────────────┼───────────┼─────────────────────
  1 │ Learn Spring Boot    │ false     │ 2026-03-01 09:00:00
  2 │ Build TaskController │ true      │ 2026-03-01 10:00:00
  3 │ Write tests          │ false     │ 2026-03-01 11:00:00
  4 │ Add JPA              │ false     │ 2026-03-02 09:00:00
  5 │ Connect PostgreSQL   │ true      │ 2026-03-02 10:00:00
```

---

# SELECT — Read Rows

```sql
-- All tasks
SELECT * FROM tasks;

-- Only completed tasks
SELECT * FROM tasks WHERE completed = true;

-- One task by ID
SELECT * FROM tasks WHERE id = 3;

-- Only specific columns
SELECT id, title, completed FROM tasks;
```

---

# INSERT — Add a Row

```sql
INSERT INTO tasks (title, description, completed)
VALUES ('Deploy to production', 'Push to server', false);
```

- You don't provide `id` — PostgreSQL assigns it (BIGSERIAL)
- You don't provide `created_at` — `DEFAULT NOW()` handles it
- Result: a new row with `id = 6`

---

# UPDATE — Modify a Row

```sql
-- Mark task 1 as complete
UPDATE tasks SET completed = true WHERE id = 1;

-- Change title and description
UPDATE tasks
SET title = 'Write MockMvc tests', description = 'All 15 tests'
WHERE id = 3;
```

⚠️ **Always use `WHERE`** — without it, every row is updated:

```sql
UPDATE tasks SET completed = true;  -- marks ALL tasks complete!
```

---

# DELETE — Remove a Row

```sql
DELETE FROM tasks WHERE id = 4;
```

⚠️ **Always use `WHERE`** — without it, all rows are deleted:

```sql
DELETE FROM tasks;  -- empties the entire table!
```

---

# Practice

Using the sample data above, write SQL to:

1. Get all tasks that are not completed
2. Get only the `title` of task with id = 5
3. Insert a new task: title = "Review PR", not completed
4. Mark task 3 as completed
5. Delete task 2

<v-click>

```sql
-- 1.
SELECT * FROM tasks WHERE completed = false;
-- 2.
SELECT title FROM tasks WHERE id = 5;
-- 3.
INSERT INTO tasks (title, completed) VALUES ('Review PR', false);
-- 4.
UPDATE tasks SET completed = true WHERE id = 3;
-- 5.
DELETE FROM tasks WHERE id = 2;
```

</v-click>
