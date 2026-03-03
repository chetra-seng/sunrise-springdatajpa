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
zoom: 0.85
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
zoom: 0.85
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

<div class="flex flex-col items-center gap-0 mt-4 select-none">

  <!-- HTTP Request -->
  <div class="text-xs font-mono text-slate-500 dark:text-slate-400 mb-1">HTTP Request</div>
  <div class="w-0.5 h-4 bg-slate-400 dark:bg-slate-500"></div>

  <!-- TaskController -->
  <div class="flex items-center gap-3">
    <div class="w-48 text-center px-4 py-2.5 rounded-lg border-2 border-green-400 dark:border-green-500 bg-green-50 dark:bg-green-900/30 font-mono text-sm font-semibold text-green-800 dark:text-green-300">
      TaskController
    </div>
    <span class="text-xs font-semibold px-2 py-0.5 rounded-full bg-green-100 dark:bg-green-900/50 text-green-700 dark:text-green-400">UNCHANGED</span>
  </div>

  <div class="w-0.5 h-4 bg-slate-400 dark:bg-slate-500"></div>

  <!-- TaskServiceImpl -->
  <div class="flex items-center gap-3">
    <div class="w-48 text-center px-4 py-2.5 rounded-lg border-2 border-green-400 dark:border-green-500 bg-green-50 dark:bg-green-900/30 font-mono text-sm font-semibold text-green-800 dark:text-green-300">
      TaskServiceImpl
    </div>
    <span class="text-xs font-semibold px-2 py-0.5 rounded-full bg-green-100 dark:bg-green-900/50 text-green-700 dark:text-green-400">UNCHANGED</span>
  </div>

  <div class="w-0.5 h-4 bg-slate-400 dark:bg-slate-500"></div>

  <!-- TaskRepository -->
  <div class="flex items-center gap-3">
    <div class="w-48 text-center px-4 py-2.5 rounded-lg border-2 border-amber-400 dark:border-amber-500 bg-amber-50 dark:bg-amber-900/30 font-mono text-sm font-semibold text-amber-800 dark:text-amber-300">
      TaskRepository
    </div>
    <span class="text-xs font-semibold px-2 py-0.5 rounded-full bg-amber-100 dark:bg-amber-900/50 text-amber-700 dark:text-amber-400">REPLACED</span>
  </div>

  <div class="w-0.5 h-4 bg-slate-400 dark:bg-slate-500"></div>

  <!-- PostgreSQL -->
  <div class="flex items-center gap-3">
    <div class="w-48 text-center px-4 py-2.5 rounded-lg border-2 border-blue-400 dark:border-blue-500 bg-blue-50 dark:bg-blue-900/30 font-mono text-sm font-semibold text-blue-800 dark:text-blue-300">
      PostgreSQL
    </div>
    <span class="text-xs font-semibold px-2 py-0.5 rounded-full bg-blue-100 dark:bg-blue-900/50 text-blue-700 dark:text-blue-400">NEW</span>
  </div>

</div>

<div class="mt-6 text-center text-sm text-slate-500 dark:text-slate-400">
  Controller and service untouched — only the repository layer swapped out.
</div>
