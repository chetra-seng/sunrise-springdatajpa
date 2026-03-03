---
layout: center
---
# Spring Data JPA

## What It Is and How It Works

---

# What is JPA?

**JPA** = Java Persistence API — a specification for mapping Java objects to database tables.

**Hibernate** = the most popular JPA implementation. It does the actual SQL generation.

**Spring Data JPA** = Spring's layer on top — makes it even easier to use.

<div class="flex flex-col items-center gap-0 mt-2 select-none">

  <div class="w-56 text-center px-3 py-1.5 rounded-lg border-2 border-blue-400 dark:border-blue-500 bg-blue-50 dark:bg-blue-900/30 font-mono text-xs font-semibold text-blue-800 dark:text-blue-300">
    Your Java Code
    <div class="text-xs font-normal text-blue-600 dark:text-blue-400">@Entity, JpaRepository</div>
  </div>

  <div class="w-0.5 h-4 bg-slate-400 dark:bg-slate-500"></div>

  <div class="w-56 text-center px-3 py-1.5 rounded-lg border-2 border-green-400 dark:border-green-500 bg-green-50 dark:bg-green-900/30 font-mono text-xs font-semibold text-green-800 dark:text-green-300">
    Spring Data JPA
    <div class="text-xs font-normal text-green-600 dark:text-green-400">repository abstraction</div>
  </div>

  <div class="w-0.5 h-4 bg-slate-400 dark:bg-slate-500"></div>

  <div class="w-56 text-center px-3 py-1.5 rounded-lg border-2 border-amber-400 dark:border-amber-500 bg-amber-50 dark:bg-amber-900/30 font-mono text-xs font-semibold text-amber-800 dark:text-amber-300">
    Hibernate
    <div class="text-xs font-normal text-amber-600 dark:text-amber-400">JPA implementation — generates SQL</div>
  </div>

  <div class="w-0.5 h-4 bg-slate-400 dark:bg-slate-500"></div>

  <div class="w-56 text-center px-3 py-1.5 rounded-lg border-2 border-slate-400 dark:border-slate-500 bg-slate-50 dark:bg-slate-800/50 font-mono text-xs font-semibold text-slate-700 dark:text-slate-300">
    JDBC Driver
    <div class="text-xs font-normal text-slate-500 dark:text-slate-400">sends SQL over the wire</div>
  </div>

  <div class="w-0.5 h-4 bg-slate-400 dark:bg-slate-500"></div>

  <div class="w-56 text-center px-3 py-1.5 rounded-lg border-2 border-indigo-400 dark:border-indigo-500 bg-indigo-50 dark:bg-indigo-900/30 font-mono text-xs font-semibold text-indigo-800 dark:text-indigo-300">
    PostgreSQL
    <div class="text-xs font-normal text-indigo-600 dark:text-indigo-400">stores and queries your data</div>
  </div>

</div>

---

# The Key Promise

You write Java. Hibernate writes SQL.

```java
// You write:
taskRepository.findAll();

// Hibernate runs:
SELECT task0_.id, task0_.title, task0_.description,
       task0_.completed, task0_.created_at
FROM tasks task0_
```

<v-click>

Enable `spring.jpa.show-sql=true` and `spring.jpa.properties.hibernate.format_sql=true`
in `application.properties` to see every query in your logs.

</v-click>

---

# show-sql in Action

With these properties set:
```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

Every database operation logs its SQL:

```
Hibernate:
    select
        t1_0.id,
        t1_0.title,
        t1_0.description,
        t1_0.completed,
        t1_0.created_at
    from
        tasks t1_0
```

This is your best tool for understanding what JPA is doing.

---

# Three Things JPA Needs

To use JPA, three things need to happen:

<v-clicks>

**1. Mark your model as a database entity:**
```java
@Entity
@Table(name = "tasks")
public class TaskModel { ... }
```

**2. Replace your manual repository with a JPA interface:**
```java
public interface TaskRepository extends JpaRepository<TaskModel, Long> { }
```

**3. Configure the database connection:**
```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/taskflow
```

</v-clicks>

---

# What JPA Gives You for Free

When `TaskRepository extends JpaRepository<TaskModel, Long>`:

```java
// These all exist without writing any code:
taskRepository.findAll()              // SELECT * FROM tasks
taskRepository.findById(id)          // SELECT * FROM tasks WHERE id = ?
taskRepository.save(task)            // INSERT or UPDATE
taskRepository.deleteById(id)        // DELETE FROM tasks WHERE id = ?
taskRepository.count()               // SELECT COUNT(*) FROM tasks
taskRepository.existsById(id)        // SELECT COUNT(*) FROM tasks WHERE id = ?
```

<v-click>

These method names match what we implemented manually in `TaskRepository.java`.
That's why the service doesn't need to change.

</v-click>

---

# From Class to Interface

````md magic-move
```java
// Before: a class you wrote yourself
@Repository
public class TaskRepository {
    private Map<Long, TaskModel> tasks = new ConcurrentHashMap<>();
    private AtomicLong counter = new AtomicLong(0);

    public List<TaskModel> findAll() { return tasks.values().stream().toList(); }
    public Optional<TaskModel> findById(Long id) { return Optional.ofNullable(tasks.get(id)); }
    public TaskModel save(TaskModel task) { /* assign id + put in map */ }
    public Boolean delete(Long id) { /* containsKey + remove */ }
}
```

```java
// After: an interface Spring implements for you
public interface TaskRepository extends JpaRepository<TaskModel, Long> {
    // All four methods above exist by default
    // Add custom queries as needed
}
```
````

Spring generates the entire implementation at startup — no `ConcurrentHashMap`, no `AtomicLong`.

---
zoom: 0.85
---

# The ddl-auto Setting

```properties
spring.jpa.hibernate.ddl-auto=update
```

Controls what happens to the schema at startup:

| Value | Behavior |
|-------|----------|
| `create` | Drop and recreate all tables |
| `create-drop` | Create on start, drop on stop |
| `update` | Add missing columns/tables, keep existing data |
| `validate` | Check schema matches entities, fail if not |
| `none` | Do nothing |

<v-click>

Use `update` in dev (safe, evolves the schema). Use `validate` or `none` in production with proper migrations (Flyway/Liquibase).

</v-click>

