---
layout: center
---
# Filtering and Sorting

## WHERE · ORDER BY · LIKE · LIMIT

---

# WHERE — Filter Rows

```sql
-- Completed tasks only
SELECT * FROM tasks WHERE completed = true;

-- Incomplete tasks only
SELECT * FROM tasks WHERE completed = false;

-- Combine conditions with AND / OR
SELECT * FROM tasks
WHERE completed = false AND title LIKE '%Spring%';
```

---

# LIKE — Text Pattern Matching

`%` is the wildcard — matches any sequence of characters.

```sql
-- Starts with "Learn"
SELECT * FROM tasks WHERE title LIKE 'Learn%';

-- Contains "spring" anywhere
SELECT * FROM tasks WHERE title LIKE '%spring%';

-- Case-insensitive match (PostgreSQL)
SELECT * FROM tasks WHERE LOWER(title) LIKE '%spring%';
```

---

# ORDER BY — Sort Results

```sql
-- Newest first
SELECT * FROM tasks ORDER BY created_at DESC;

-- Oldest first
SELECT * FROM tasks ORDER BY created_at ASC;

-- Alphabetical by title
SELECT * FROM tasks ORDER BY title ASC;
```

`ASC` = ascending (default), `DESC` = descending.

---

# LIMIT and OFFSET — Pagination

```sql
-- First 3 tasks
SELECT * FROM tasks ORDER BY created_at DESC LIMIT 3;

-- Tasks 4–6 (second page)
SELECT * FROM tasks ORDER BY created_at DESC LIMIT 3 OFFSET 3;
```

`LIMIT` caps the number of rows returned. `OFFSET` skips the first N rows.

---

# Aggregate Functions

```sql
-- How many tasks exist?
SELECT COUNT(*) FROM tasks;

-- How many are completed?
SELECT COUNT(*) FROM tasks WHERE completed = true;
```

---

# Practice

Write SQL for each:

1. All tasks with "JPA" in the title (case-insensitive)
2. Incomplete tasks, ordered oldest first
3. The 2 most recently created tasks
4. Count how many tasks are not completed

<v-click>

```sql
-- 1.
SELECT * FROM tasks WHERE LOWER(title) LIKE '%jpa%';
-- 2.
SELECT * FROM tasks WHERE completed = false ORDER BY created_at ASC;
-- 3.
SELECT * FROM tasks ORDER BY created_at DESC LIMIT 2;
-- 4.
SELECT COUNT(*) FROM tasks WHERE completed = false;
```

</v-click>
