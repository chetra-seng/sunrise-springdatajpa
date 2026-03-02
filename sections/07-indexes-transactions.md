---
layout: center
---
# Transactions and Indexes

---

# What is a Transaction?

A **transaction** groups SQL statements so they either all succeed or all fail together.

```sql
BEGIN;

  UPDATE tasks SET completed = true WHERE id = 1;
  INSERT INTO audit_log (task_id, action) VALUES (1, 'completed');

COMMIT;   -- both changes saved
```

If anything fails between `BEGIN` and `COMMIT`:

```sql
ROLLBACK;   -- neither change is saved
```

---

# Why Transactions Matter

Without a transaction, a crash mid-way leaves your data inconsistent:

```sql
-- Without transaction:
UPDATE tasks SET completed = true WHERE id = 5;
-- server crashes here
UPDATE projects SET done_count = done_count + 1 WHERE id = 3;
-- ← never runs — task is complete but count is wrong
```

```sql
-- With transaction:
BEGIN;
  UPDATE tasks SET completed = true WHERE id = 5;
  UPDATE projects SET done_count = done_count + 1 WHERE id = 3;
COMMIT;   -- both or neither
```

---

# What is an Index?

An **index** lets the database find rows fast without scanning every row — like an index at the back of a book.

```sql
-- Without index: scans all rows to find completed = true
SELECT * FROM tasks WHERE completed = true;

-- Create an index on the completed column
CREATE INDEX idx_tasks_completed ON tasks(completed);

-- Now the same query jumps directly to matching rows
SELECT * FROM tasks WHERE completed = true;
```

---

# When to Add an Index

Add indexes on columns used in `WHERE`, `JOIN`, or `ORDER BY`:

```sql
-- These queries benefit from indexes:
SELECT * FROM tasks WHERE completed = true;          -- index on completed
SELECT * FROM tasks WHERE project_id = 1;            -- index on project_id
SELECT * FROM tasks ORDER BY created_at DESC;        -- index on created_at
```

Primary keys (`id`) are indexed automatically. Foreign key columns (`project_id`) are not — add them manually.

---

# Index Trade-offs

Indexes are not free:

| | Reads | Writes |
|-|-------|--------|
| With index | Fast — skips irrelevant rows | Slower — index must be updated |
| Without index | Slow at scale — full table scan | Fast |

Add indexes on columns you filter or sort by often. Don't index every column.

---

# SQL Review Complete

You now know the SQL that every web application runs:

| Operation | SQL |
|-----------|-----|
| Read all | `SELECT * FROM tasks` |
| Read with filter | `SELECT * FROM tasks WHERE completed = true` |
| Read across tables | `SELECT ... FROM tasks JOIN projects ON ...` |
| Create | `INSERT INTO tasks (title, ...) VALUES (...)` |
| Update | `UPDATE tasks SET completed = true WHERE id = ?` |
| Delete | `DELETE FROM tasks WHERE id = ?` |
| Performance | `CREATE INDEX idx ON tasks(completed)` |
| Integrity | `BEGIN ... COMMIT / ROLLBACK` |

Next: Spring Data JPA — where Java writes all of this for you.
