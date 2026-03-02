---
layout: center
---
# Databases and PostgreSQL

---

# What is a Database?

A **database** is an organized system for storing, retrieving, and managing data — designed to survive restarts, handle many users at once, and answer questions efficiently.

**Before databases**, applications stored data in files:

```
tasks.txt:
1,Learn Spring Boot,Complete the course,false,2026-03-01
2,Build TaskController,All 7 endpoints,true,2026-03-01
```

Problems:
- No way to ask "give me all incomplete tasks" — you read the whole file
- Two users writing at the same time → corrupted data
- No protection against bad data

---

# What a Database Gives You

| Problem with files | Database solution |
|--------------------|------------------|
| Slow to search | Indexes — find data in milliseconds |
| Concurrent writes corrupt data | Transactions — safe concurrent access |
| No data validation | Constraints — `NOT NULL`, `BOOLEAN`, type checks |
| Lost on restart | Persisted to disk |
| Can't link data across files | Relationships — foreign keys |

---

# What is a Relational Database?

A **relational database** organizes data into **tables** that can reference each other.

```
projects table:              tasks table:
┌────┬───────────────┐       ┌────┬──────────────────────┬────────────┐
│ id │ name          │       │ id │ title                │ project_id │
├────┼───────────────┤       ├────┼──────────────────────┼────────────┤
│  1 │ Sunrise Course│       │  1 │ Learn Spring Boot    │     1      │
│  2 │ Personal      │       │  2 │ Build TaskController │     1      │
└────┴───────────────┘       │  3 │ Buy groceries        │     2      │
                             └────┴──────────────────────┴────────────┘
```

`project_id` in tasks **references** the `id` in projects. That link is the **relationship**.

---

# Tables — Rows and Columns

```
tasks table:
┌────┬──────────────────────┬─────────────────────┬───────────┬─────────────────────┐
│ id │ title                │ description         │ completed │ created_at          │
├────┼──────────────────────┼─────────────────────┼───────────┼─────────────────────┤
│  1 │ Learn Spring Boot    │ Complete the course │ false     │ 2026-03-01 10:00:00 │
│  2 │ Build TaskController │ All 7 endpoints     │ true      │ 2026-03-01 11:30:00 │
│  3 │ Write tests          │ MockMvc tests       │ false     │ 2026-03-02 09:00:00 │
└────┴──────────────────────┴─────────────────────┴───────────┴─────────────────────┘
```

- **table** → stores one type of thing
- **row** → one record
- **column** → one attribute, with a fixed type

---

# Schema — Enforced Structure

Unlike a spreadsheet, a database enforces the shape of your data:

```sql
CREATE TABLE tasks (
    id          BIGSERIAL    PRIMARY KEY,
    title       VARCHAR(255) NOT NULL,
    description TEXT,
    completed   BOOLEAN      NOT NULL DEFAULT false,
    created_at  TIMESTAMP    NOT NULL DEFAULT NOW()
);
```

- `BIGSERIAL` → must be an integer, auto-assigned
- `VARCHAR(255)` → string, max 255 characters
- `NOT NULL` → this column cannot be empty
- `BOOLEAN` → can only be `true` or `false`
- `DEFAULT false` → use this value if none is provided

---

# What is PostgreSQL?

**PostgreSQL** (Postgres) is a free, open-source relational database — one of the most widely used in the world.

<v-clicks>

- Started at UC Berkeley in 1986, open source since 1996
- Used by Apple, Instagram, Spotify, Reddit, and many others
- Follows the SQL standard closely
- Adds extras: JSON columns, full-text search, geographic data
- The default choice for Java/Spring Boot production apps

</v-clicks>

---

# PostgreSQL Architecture

```
PostgreSQL server  (port 5432)
  │
  ├── Database: taskflow          ← one database per application
  │     └── Schema: public
  │           ├── Table: tasks
  │           ├── Table: projects
  │           └── Table: comments
  │
  └── Database: other_app         ← isolated from taskflow
```

Spring Boot connects to **one specific database** per application.

---

# Connecting to PostgreSQL

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/taskflow
spring.datasource.username=postgres
spring.datasource.password=secret
```

Breaking down the URL:

```
jdbc:postgresql://localhost:5432/taskflow
                  │         │    └── database name
                  │         └── port (PostgreSQL default)
                  └── host
```

---

# PostgreSQL vs Other Databases

| Database | Type | Common use |
|----------|------|-----------|
| **PostgreSQL** | Relational | Production apps — our choice |
| **MySQL / MariaDB** | Relational | Also common for web apps |
| **H2** | Relational (in-memory) | Tests only — no setup needed |
| **SQLite** | Relational (file) | Mobile, local tools |
| **MongoDB** | Document (JSON) | Schema-less data |
| **Redis** | Key-value | Caching, sessions |
