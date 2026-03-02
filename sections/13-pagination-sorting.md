---
layout: center
---
# Migrating the Task API

## Step-by-Step: Task.java and TaskRepository.java

---

# The Migration Plan

| File | Change |
|------|--------|
| `pom.xml` | ✅ Added JPA + PostgreSQL deps (previous section) |
| `application.properties` | ✅ Added datasource + JPA config (previous section) |
| `Task.java` | Add `@Entity`, `@Id`, `@GeneratedValue`, `@CreationTimestamp` |
| `TaskRepository.java` | Replace class with interface extending `JpaRepository` |
| `TaskServiceImpl.java` | Minor adjustment to `delete()` — everything else unchanged |
| `TaskController.java` | **No changes** |
| `TaskMapper.java` | **No changes** |

---

# Step 1: Update Task.java

Open `Task.java` and make these changes:

```java
package com.chetraseng.sunrise_task_flow_api.model;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;
import java.time.LocalDateTime;

@Entity
@Table(name = "tasks")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Task {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private String description;
    private Boolean completed = false;

    @CreationTimestamp
    private LocalDateTime createdAt;   // ← remove = LocalDateTime.now()
}
```

---

# Task.java — What Changed

| Before | After |
|--------|-------|
| No annotations | `@Entity @Table(name = "tasks")` |
| `private Long id` | `@Id @GeneratedValue(IDENTITY) private Long id` |
| `private LocalDateTime createdAt = LocalDateTime.now()` | `@CreationTimestamp private LocalDateTime createdAt` |
| No `@Builder` | Added `@Builder` |

The five fields are the same. The types are the same. Only annotations added.

---

# Step 2: Replace TaskRepository.java

**Delete** the entire class body. Replace with:

```java
package com.chetraseng.sunrise_task_flow_api.repository;

import com.chetraseng.sunrise_task_flow_api.model.Task;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface TaskRepository extends JpaRepository<Task, Long> {

    List<Task> findByCompleted(boolean completed);

}
```

That's the entire file. No `ConcurrentHashMap`. No `AtomicLong`. No `@Repository`.

---

# Step 3: Update TaskServiceImpl.java — delete()

The only method that needs updating is `delete()`.

Our old `TaskRepository.delete(id)` returned `Boolean`. JpaRepository's `deleteById()` does not.

```java
// Before:
@Override
public boolean delete(Long id) {
    return taskRepository.delete(id);   // returned Boolean
}

// After:
@Override
public boolean delete(Long id) {
    if (!taskRepository.existsById(id)) {
        return false;
    }
    taskRepository.deleteById(id);
    return true;
}
```

---

# Step 4: Update TaskServiceImpl.java — findAll()

Optionally push the `completed` filter to the database:

```java
// The current service signature:
public List<TaskResponse> findAll() {
    return taskRepository.findAll().stream()
        .map(taskMapper::toTaskResponse).toList();
}
```

The `?completed=` filter currently runs in the controller as a stream filter.
You can keep it there — or add a new service method:

```java
public List<TaskResponse> findByCompleted(boolean completed) {
    return taskRepository.findByCompleted(completed).stream()
        .map(taskMapper::toTaskResponse).toList();
}
```

Both approaches work. Moving the filter to the DB is better at scale.

---

# Step 5: Verify — Run the Tests

```bash
./mvnw test
```

The existing `TaskControllerTest` tests should pass. With H2 in test scope and the test `application.properties` configured:
- Each test run creates a fresh schema
- No `@DirtiesContext` needed — JPA + H2 handles isolation via `@Transactional`

<v-click>

If tests fail with `Table "TASKS" not found`, check that `src/test/resources/application.properties` exists and has the H2 config.

</v-click>

---

# What the Migration Looks Like Live

Enable `show-sql=true` and run:

```bash
curl -X POST http://localhost:9999/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"First DB task","description":"Stored in PostgreSQL"}'
```

Log output:
```sql
Hibernate:
    insert
    into
        tasks (completed, created_at, description, title)
    values
        (?, ?, ?, ?)
```

```json
{"id":1,"title":"First DB task","description":"Stored in PostgreSQL",
 "completed":false,"createdAt":"2026-03-02T10:00:00"}
```

---

# Architecture: What Changed vs Stayed the Same

```
HTTP Request
     │
     ▼
┌─────────────────┐
│  TaskController  │  ← UNCHANGED
└────────┬─────────┘
         │
         ▼
┌─────────────────┐
│ TaskServiceImpl  │  ← one method updated (delete)
└────────┬─────────┘
         │
         ▼
┌─────────────────┐
│ TaskRepository   │  ← REPLACED: class → interface
└────────┬─────────┘
         │
         ▼
┌─────────────────┐
│   PostgreSQL     │  ← NEW
└─────────────────┘
```

The controller and mapper are completely untouched. One repository file replaced.
