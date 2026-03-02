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

```
Your Java code (@Entity, JpaRepository)
         ↓
   Spring Data JPA
         ↓
      Hibernate
         ↓
    JDBC Driver
         ↓
    PostgreSQL
```

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
public class Task { ... }
```

**2. Replace your manual repository with a JPA interface:**
```java
public interface TaskRepository extends JpaRepository<Task, Long> { }
```

**3. Configure the database connection:**
```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/taskflow
```

</v-clicks>

---

# What JPA Gives You for Free

When `TaskRepository extends JpaRepository<Task, Long>`:

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

# Comparing the Two Approaches

```java
// Before (manual ConcurrentHashMap):
@Repository
public class TaskRepository {
    private Map<Long, Task> tasks = new ConcurrentHashMap<>();
    private AtomicLong counter = new AtomicLong(0);

    public List<Task> findAll() { return tasks.values().stream().toList(); }
    public Optional<Task> findById(Long id) { return Optional.ofNullable(tasks.get(id)); }
    public Task save(Task task) { /* assign id + put in map */ }
    public Boolean delete(Long id) { /* containsKey + remove */ }
}

// After (Spring Data JPA):
public interface TaskRepository extends JpaRepository<Task, Long> {
    // All four methods above exist by default
    // Add custom queries as needed
}
```

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

---

# The Learning Path

```
SQL (done): SELECT, INSERT, JOINs, transactions
     ↓
JPA (now):  @Entity, JpaRepository, derived queries
     ↓
Migration:  swap TaskRepository, update Task.java
     ↓
Relationships: @OneToMany, @ManyToMany
```

Every JPA concept we learn maps directly to SQL we already understand.
