---
layout: center
---
# Transactions & Performance

## @Transactional, N+1 Queries, and FetchType

---

# What Is a Transaction?

A transaction is a group of DB operations that succeed or fail together.

```sql
BEGIN;
UPDATE tasks SET completed = true WHERE id = 5;
UPDATE tasks SET updated_at = NOW() WHERE id = 5;
COMMIT;   -- both changes saved
-- or:
ROLLBACK; -- both changes undone if anything fails
```

<v-click>

Without transactions, if the second `UPDATE` crashes, you end up with `completed = true` but no `updated_at` — **inconsistent data**.

</v-click>

---

# @Transactional in Spring

Spring wraps a method in a transaction automatically:

```java
@Service
@RequiredArgsConstructor
public class TaskServiceImpl implements TaskService {

    private final TaskRepository taskRepository;

    @Transactional          // ← Spring opens a transaction before this method
    public TaskResponse complete(Long id) {
        TaskModel task = taskRepository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));

        task.setCompleted(true);
        return taskMapper.toTaskResponse(taskRepository.save(task));
    }   // ← Spring commits here; if exception thrown → auto rollback
}
```

If anything inside `complete()` throws a `RuntimeException`, Spring rolls back the entire transaction.

---

# Where to Put @Transactional

```java
// ✅ On the service method — correct place:
@Service
public class TaskServiceImpl {

    @Transactional
    public TaskResponse update(Long id, TaskRequest request) { ... }

    @Transactional
    public boolean delete(Long id) { ... }
}
```

```java
// ❌ Don't put @Transactional on the controller:
@RestController
public class TaskController {

    @Transactional   // wrong layer — controllers shouldn't own transactions
    @PutMapping("/{id}")
    public ResponseEntity<TaskResponse> update(...) { ... }
}
```

**Rule:** transactions belong on the service layer.

---

# @Transactional(readOnly = true)

For read-only methods, you can hint to the DB that no writes will happen:

```java
@Transactional(readOnly = true)
public List<TaskResponse> findAll() {
    return taskRepository.findAll().stream()
        .map(taskMapper::toTaskResponse).toList();
}

@Transactional(readOnly = true)
public TaskResponse findById(Long id) {
    return taskRepository.findById(id)
        .map(taskMapper::toTaskResponse)
        .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
}
```

Benefits:
- Hibernate skips **dirty checking** (no need to track entity changes)
- Some databases optimize read-only transactions (e.g., replicas, snapshots)

---

# The N+1 Query Problem

**Scenario:** Load 10 tasks, each with a project. How many SQL queries run?

```java
List<TaskModel> tasks = taskRepository.findAll();  // 1 query

for (TaskModel task : tasks) {
    System.out.println(task.getProject().getName());  // 1 query PER task
}
```

With `show-sql=true`, you'd see:

```sql
SELECT * FROM tasks;                        -- 1 query
SELECT * FROM projects WHERE id = 1;        -- for task 1
SELECT * FROM projects WHERE id = 2;        -- for task 2
SELECT * FROM projects WHERE id = 3;        -- for task 3
-- ... 10 more queries for 10 tasks
```

**1 + 10 = 11 queries** to load 10 tasks. This is the N+1 problem.

---

# FetchType: LAZY vs EAGER

```java
// Default for @ManyToOne — EAGER (loads immediately):
@ManyToOne(fetch = FetchType.EAGER)
@JoinColumn(name = "project_id")
private ProjectModel project;
// → Always JOINs project when loading task

// LAZY — load only when accessed:
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "project_id")
private ProjectModel project;
// → Loads project only if you call task.getProject()
```

| | EAGER | LAZY |
|-|-------|------|
| When | On every load | On first access |
| Extra queries? | No (joined) | Yes (if accessed) |
| Risk | Always fetches even when unused | N+1 if accessed in a loop |

---
zoom: 0.85
---

# Solving N+1 with JOIN FETCH

Use `@Query` with `JOIN FETCH` to load relationships in one query:

```java
public interface TaskRepository extends JpaRepository<TaskModel, Long> {

    @Query("SELECT t FROM TaskModel t LEFT JOIN FETCH t.project")
    List<TaskModel> findAllWithProject();

}
```

```sql
-- One query instead of N+1:
SELECT t.*, p.*
FROM tasks t
LEFT JOIN projects p ON t.project_id = p.id
```

<v-click>

Use `show-sql=true` to **confirm** how many queries your code runs. If you see the same table being queried in a loop, you have an N+1 problem.

</v-click>

---

# Using show-sql to Diagnose

In `application.properties`:
```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

Then run your endpoint and read the logs:

```sql
-- GOOD: one query with a join
Hibernate:
    select t.id, t.title, t.completed, p.id, p.name
    from tasks t left join projects p on t.project_id = p.id

-- BAD: N+1
Hibernate: select * from tasks
Hibernate: select * from projects where id=?
Hibernate: select * from projects where id=?
Hibernate: select * from projects where id=?
```

If you see the same query repeated, fix it with `JOIN FETCH`.

---

# @Transactional + Lazy Loading Trap

```java
// ❌ This fails with LazyInitializationException:
public TaskResponse findById(Long id) {
    TaskModel task = taskRepository.findById(id).orElseThrow();
    // Transaction closes here after findById returns
    return task.getProject().getName();  // BOOM — session already closed
}

// ✅ Keep @Transactional open for the whole method:
@Transactional(readOnly = true)
public TaskResponse findById(Long id) {
    TaskModel task = taskRepository.findById(id).orElseThrow();
    return task.getProject().getName();  // ✓ session still open
}
```

`LazyInitializationException` = you tried to access a lazy relationship after the Hibernate session closed.

---

# Summary

| Concept | What It Does |
|---------|-------------|
| `@Transactional` | Wraps method in a DB transaction; auto-rollback on exception |
| `@Transactional(readOnly = true)` | Optimized for read-only methods — skips dirty checking |
| `FetchType.LAZY` | Load relationship only when accessed |
| `FetchType.EAGER` | Always JOIN relationship when loading entity |
| N+1 problem | 1 query for list + N queries for each row's relationship |
| `JOIN FETCH` | Fix N+1 by loading relationship in one query |
| `show-sql=true` | See the SQL Hibernate generates — your best debugging tool |
