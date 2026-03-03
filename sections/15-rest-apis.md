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
public ResponseEntity<Task> getTask(@PathVariable Long id) {
    return ResponseEntity.ok(taskRepository.findById(id).orElseThrow());
}
```

Problems:
- Exposes internal DB column names and lazy relationships
- Can trigger `LazyInitializationException` during JSON serialization
- If Task has a `Project` with `List<Task>`, Jackson serializes **infinitely**

---

# The DTO Pattern (Already in Your Project)

You already use DTOs via MapStruct. This is the right pattern:

```java
// ✅ Controller returns DTO, not entity:
@GetMapping("/{id}")
public ResponseEntity<TaskResponse> getTask(@PathVariable Long id) {
    Task task = taskRepository.findById(id)
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
// Project has List<Task> tasks
// Task has Project project
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
    // No List<Task> — if you need tasks, add a separate endpoint
}
```

---

# Nested DTOs for Relationships

When the client needs related data, include it in the response DTO:

```java
// TaskResponse that includes project name (not the full Project entity):
@Data
@NoArgsConstructor
@AllArgsConstructor
public class TaskResponse {
    private Long id;
    private String title;
    private String description;
    private Boolean completed;
    private LocalDateTime createdAt;
    private Long projectId;       // just the FK — no Project object
    private String projectName;   // denormalized for display convenience
}
```

MapStruct can map `task.project.id` and `task.project.name` automatically:

```java
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface TaskMapper {

    @Mapping(target = "projectId", source = "project.id")
    @Mapping(target = "projectName", source = "project.name")
    TaskResponse toTaskResponse(Task task);
}
```

---
zoom: 0.85
---

# Handling 404 the Right Way

When an entity is not found, throw a `ResponseStatusException` — Spring converts it to the right HTTP status automatically:

```java
// In TaskServiceImpl.java:
@Transactional(readOnly = true)
public TaskResponse findById(Long id) {
    return taskRepository.findById(id)
        .map(taskMapper::toTaskResponse)
        .orElseThrow(() -> new ResponseStatusException(
            HttpStatus.NOT_FOUND, "Task not found: " + id
        ));
}
```

```bash
curl http://localhost:9999/api/tasks/999
# → HTTP 404
# {"status":404,"error":"Not Found","message":"Task not found: 999"}
```

---

# Request DTO Validation

Your `TaskRequest` DTO is the entry point for client data — validate it:

```java
// pom.xml — add:
// spring-boot-starter-validation

@Data
@NoArgsConstructor
@AllArgsConstructor
public class TaskRequest {

    @NotBlank(message = "Title is required")
    @Size(max = 255, message = "Title must be 255 characters or less")
    private String title;

    private String description;
}
```

```java
// In controller:
@PostMapping
public ResponseEntity<TaskResponse> create(@Valid @RequestBody TaskRequest request) { ... }
```

Validation happens before the request reaches the service or database.

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
