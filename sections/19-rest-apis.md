---
layout: center
---
# REST APIs & JPA

## DTOs, Serialization, and Avoiding Common Pitfalls

---

# Why DTOs Matter Even More With JPA

With in-memory storage, returning your model directly was low-risk.

With JPA entities, returning them directly causes problems:

```java
// ❌ Returning the entity directly:
@GetMapping("/{id}")
public ResponseEntity<TaskModel> getTask(@PathVariable Long id) {
    return ResponseEntity.ok(taskRepository.findById(id).orElseThrow());
}
```

Problems:
- Exposes internal DB column names and lazy relationships
- Can trigger `LazyInitializationException` during JSON serialization
- If Task has a `ProjectModel` with `List<TaskModel>`, Jackson serializes **infinitely**

---

# The DTO Pattern (Already in Your Project)

You already use DTOs via MapStruct. This is the right pattern:

```java
// ✅ Controller returns DTO, not entity:
@GetMapping("/{id}")
public ResponseEntity<TaskResponse> getTask(@PathVariable Long id) {
    TaskModel task = taskRepository.findById(id)
        .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
    return ResponseEntity.ok(taskMapper.toTaskResponse(task));
}
```

`TaskResponse` is a plain POJO — no JPA proxies, no lazy-loading triggers, safe to serialize.

---
zoom: 0.85
---

# What TaskResponse Looks Like

```java
// Your existing TaskResponse.java:
@Data
@NoArgsConstructor
@AllArgsConstructor
public class TaskResponse {
    private Long id;
    private String title;
    private String description;
    private Boolean completed;
    private LocalDateTime createdAt;
}
```

This has **no JPA annotations**, no entity references, no lazy relationships.

Jackson serializes this cleanly every time.

---

# Infinite Recursion Problem

Once you add relationships (Project → Task → Project → Task...), returning entities breaks:

```java
// Project has List<TaskModel> tasks
// Task has ProjectModel project
// Jackson follows the graph → StackOverflowError
```

Even with `@JsonIgnore` it's fragile. The real fix: **always use DTOs**.

```java
// ✅ ProjectResponse DTO — no Task references:
@Data
@NoArgsConstructor
@AllArgsConstructor
public class ProjectResponse {
    private Long id;
    private String name;
    private String description;
    private LocalDateTime createdAt;
    // No List<TaskModel> — if you need tasks, add a separate endpoint
}
```

---

# Nested DTOs for Relationships

When the client needs related data, include it flat in the response DTO — not as a nested entity:

```java
// TaskResponse — includes project fields, not the ProjectModel entity:
@Data
@NoArgsConstructor
@AllArgsConstructor
public class TaskResponse {
    private Long id;
    private String title;
    private String description;
    private Boolean completed;
    private LocalDateTime createdAt;
    private Long projectId;       // just the FK — no ProjectModel object
    private String projectName;   // flattened for display convenience
}
```

No `ProjectModel` reference — nothing for Jackson to follow into an infinite loop.

---

# Nested DTOs — MapStruct Mapping

MapStruct traverses the relationship automatically using dot notation:

```java
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface TaskMapper {

    @Mapping(target = "projectId",   source = "project.id")
    @Mapping(target = "projectName", source = "project.name")
    TaskResponse toTaskResponse(TaskModel task);

}
```

`source = "project.id"` tells MapStruct: go into `task.getProject()`, then call `.getId()`.

```json
{
  "id": 1,
  "title": "Set up DB",
  "projectId": 5,
  "projectName": "Task Flow API"
}
```

---

# Handling 404 — Custom Exception

Define a custom exception in `exception/ResourceNotFoundException.java`:

```java
package com.chetraseng.sunrise_task_flow_api.exception;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

Throw it from the service — no HTTP knowledge needed here:

```java
// In TaskServiceImpl.java:
@Transactional(readOnly = true)
public TaskResponse findById(Long id) {
    return taskRepository.findById(id)
        .map(taskMapper::toTaskResponse)
        .orElseThrow(() -> new ResourceNotFoundException("Task not found: " + id));
}
```

---
zoom: 0.85
---

# Handling 404 — GlobalExceptionHandler

`ErrorResponse.java` — the shape of every error response:

```java
public class ErrorResponse {
    private final int status;
    private final String message;
    private final LocalDateTime timestamp;
    // constructor + getters
}
```

`GlobalExceptionHandler.java` — catches exceptions across all controllers:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleResourceNotFoundException(ResourceNotFoundException ex) {
        return new ErrorResponse(HttpStatus.NOT_FOUND.value(), ex.getMessage(), LocalDateTime.now());
    }

}
```

---

# Handling 404 — How It Flows

```
Service throws ResourceNotFoundException("Task not found: 999")
         ↓
GlobalExceptionHandler catches it
         ↓
Returns ErrorResponse(404, "Task not found: 999", timestamp)
```

```bash
curl http://localhost:9999/api/tasks/999
```
```json
{
  "status": 404,
  "message": "Task not found: 999",
  "timestamp": "2026-03-03T10:00:00"
}
```

The service only knows about domain errors. The handler only knows about HTTP. Clean separation.

---
zoom: 0.85
---

# Endpoint Design with JPA

Consistent URL patterns for related resources:

```
GET    /api/tasks                    → list all tasks (with optional ?completed=)
GET    /api/tasks/{id}               → get task by ID
POST   /api/tasks                    → create task
PUT    /api/tasks/{id}               → update task
PATCH  /api/tasks/{id}/complete      → mark as complete
DELETE /api/tasks/{id}               → delete task

GET    /api/projects                 → list projects
GET    /api/projects/{id}            → get project
POST   /api/projects                 → create project
GET    /api/projects/{id}/tasks      → tasks for a project (nested resource)

POST   /api/tasks/{id}/comments      → add comment to task
GET    /api/tasks/{id}/comments      → list comments for task
```

Nested routes (`/projects/{id}/tasks`) are natural when the child resource only makes sense in context of the parent.

---

# REST + JPA Integration Checklist

<v-clicks>

- **Never return JPA entities directly** from controllers — always use DTOs
- **Use MapStruct** to map entity → DTO (you already have `TaskMapper`)
- **@Transactional(readOnly = true)** on all GET service methods
- **@Transactional** on all write service methods (create, update, delete)
- **ResponseStatusException** for 404/400 — Spring maps it to HTTP automatically
- **show-sql=true** in dev to verify the SQL your endpoints generate
- **Page&lt;T&gt;** for list endpoints that could return many rows

</v-clicks>
