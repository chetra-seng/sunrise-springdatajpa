---
layout: center
---
# Foreign Keys and JOINs

## Relationships Between Tables

---

# The Problem: Isolated Data

Without relationships, you have no way to ask "which project does this task belong to?"

A **foreign key** is a column that points to a row in another table.

```sql
CREATE TABLE tasks (
    id          BIGSERIAL PRIMARY KEY,
    title       VARCHAR(255) NOT NULL,
    completed   BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    project_id  BIGINT REFERENCES projects(id)   -- ← foreign key
);
```

`project_id REFERENCES projects(id)` means:
- The value must exist in `projects.id` (or be `NULL`)
- Inserting a task with `project_id = 99` fails if project 99 doesn't exist

---

# What the Data Looks Like

```
projects:                        tasks:
 id │ name                        id │ title                    │ project_id
────┼────────────────            ────┼──────────────────────────┼────────────
  1 │ Sunrise Course               1 │ Complete TaskController  │     1
  2 │ Personal                     2 │ Pass all tests           │     1
                                   3 │ Buy groceries            │     2
```

Row 1 and 2 in tasks → `project_id = 1` → they belong to "Sunrise Course"

---

# JOIN — Query Across Tables

Get tasks with their project name:

```sql
SELECT t.id, t.title, t.completed, p.name AS project_name
FROM tasks t
JOIN projects p ON t.project_id = p.id;
```

```
 id │ title                    │ completed │ project_name
────┼──────────────────────────┼───────────┼────────────────
  1 │ Complete TaskController  │ false     │ Sunrise Course
  2 │ Pass all tests           │ true      │ Sunrise Course
  3 │ Buy groceries            │ false     │ Personal
```

`ON t.project_id = p.id` is the join condition — connects the FK to the PK it references.

---

# Filter With WHERE After a JOIN

```sql
-- All tasks for project 1
SELECT t.*
FROM tasks t
JOIN projects p ON t.project_id = p.id
WHERE p.id = 1;

-- All incomplete tasks with their project names
SELECT t.title, t.completed, p.name AS project_name
FROM tasks t
JOIN projects p ON t.project_id = p.id
WHERE t.completed = false;
```

---

# Multiple JOINs

```sql
-- tasks with their project AND comments
SELECT t.title, p.name AS project, c.content AS comment
FROM tasks t
JOIN projects p ON t.project_id = p.id
JOIN comments c ON c.task_id = t.id;
```

Each `JOIN` adds another table into the result. The `ON` clause is what links them.

---

# LEFT JOIN — Include Rows With No Match

```sql
-- All tasks, even those with no comments
SELECT t.title, c.content
FROM tasks t
LEFT JOIN comments c ON c.task_id = t.id;
```

```
 title                    │ content
──────────────────────────┼─────────────────
 Complete TaskController  │ Started this one
 Complete TaskController  │ Almost done
 Pass all tests           │ NULL              ← task exists, no comments
 Buy groceries            │ NULL
```

`LEFT JOIN` keeps all rows from the left table even when there's no matching right row.

---

# Practice

Use these three tables:
```
projects: id, name
tasks:    id, title, completed, project_id (FK → projects.id)
comments: id, content, task_id (FK → tasks.id)
```

Write SQL to:

1. Get all tasks belonging to the project named "Personal"
2. Get all comments for the task with id = 2, including the task title
3. Get all projects and how many tasks each has (hint: `COUNT`, `GROUP BY`)

<v-click>

```sql
-- 1.
SELECT t.* FROM tasks t
JOIN projects p ON t.project_id = p.id
WHERE p.name = 'Personal';

-- 2.
SELECT t.title, c.content FROM comments c
JOIN tasks t ON c.task_id = t.id
WHERE t.id = 2;

-- 3.
SELECT p.name, COUNT(t.id) AS task_count
FROM projects p
LEFT JOIN tasks t ON t.project_id = p.id
GROUP BY p.id, p.name;
```

</v-click>
