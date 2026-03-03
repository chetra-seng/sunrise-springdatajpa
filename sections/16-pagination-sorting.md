---
layout: center
---
# Pagination & Sorting

## Pageable, PageRequest, Page&lt;T&gt;, Sort

---

# Why Pagination Matters

Without pagination:
```java
taskRepository.findAll()
// → SELECT * FROM tasks
// Returns ALL rows — fine with 10 tasks, slow with 100,000
```

With pagination:
```java
taskRepository.findAll(PageRequest.of(0, 10))
// → SELECT * FROM tasks LIMIT 10 OFFSET 0
// Returns exactly 10 rows at a time
```

<v-click>

Pagination keeps your API fast as data grows. `Pageable` is Spring Data's standard way to pass page size and offset to any repository method.

</v-click>

---

# PageRequest.of()

```java
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;

// Page 0, 10 items per page:
Pageable pageable = PageRequest.of(0, 10);

// Page 1 (second page), 10 items:
Pageable pageable = PageRequest.of(1, 10);

// Page 0, 5 items, sorted by createdAt descending:
Pageable pageable = PageRequest.of(0, 5, Sort.by("createdAt").descending());
```

`PageRequest.of(page, size)` — page is **0-indexed**.

---
zoom: 0.85
---

# JpaRepository + Pageable

`JpaRepository` supports `Pageable` out of the box:

```java
// Returns a Page<TaskModel> — not just a List
Page<TaskModel> page = taskRepository.findAll(pageable);

// Page<T> gives you:
page.getContent()        // List<TaskModel> — the current page's rows
page.getTotalElements()  // long — total rows in the DB
page.getTotalPages()     // int  — total number of pages
page.getNumber()         // int  — current page number (0-indexed)
page.isLast()            // boolean — is this the last page?
```

```sql
-- JPA runs two queries:
SELECT * FROM tasks LIMIT 10 OFFSET 0;
SELECT COUNT(*) FROM tasks;
```

---
zoom: 0.85
---

# Paginated Endpoint Example

```java
// TaskController.java — add a paginated endpoint:
@GetMapping("/paged")
public ResponseEntity<Page<TaskResponse>> getTasksPaged(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {

    Pageable pageable = PageRequest.of(page, size);

    Page<TaskResponse> result = taskRepository.findAll(pageable)
        .map(taskMapper::toTaskResponse);   // map() works on Page<T>

    return ResponseEntity.ok(result);
}
```

```bash
curl http://localhost:9999/api/tasks/paged?page=0&size=5
```

---

# Page&lt;T&gt; JSON Response

```json
{
  "content": [
    {"id": 1, "title": "Set up PostgreSQL", "completed": false},
    {"id": 2, "title": "Add @Entity to Task", "completed": true}
  ],
  "totalElements": 42,
  "totalPages": 9,
  "number": 0,
  "size": 5,
  "last": false,
  "first": true
}
```

Clients use `totalPages` and `number` to build next/previous navigation.

---

# Sorting with Sort

```java
import org.springframework.data.domain.Sort;

// Sort by one field:
Sort sort = Sort.by("createdAt").descending();
taskRepository.findAll(sort);
// → SELECT * FROM tasks ORDER BY created_at DESC

// Sort by multiple fields:
Sort sort = Sort.by("completed").ascending()
               .and(Sort.by("createdAt").descending());
// → ORDER BY completed ASC, created_at DESC
```

---

# Sorting Without Pagination

```java
// Use Sort directly — no page wrapping:
List<TaskModel> tasks = taskRepository.findAll(Sort.by("title").ascending());
// → SELECT * FROM tasks ORDER BY title ASC

// In controller:
@GetMapping
public ResponseEntity<List<TaskResponse>> getTasks(
        @RequestParam(defaultValue = "createdAt") String sortBy,
        @RequestParam(defaultValue = "desc") String direction) {

    Sort sort = direction.equalsIgnoreCase("asc")
        ? Sort.by(sortBy).ascending()
        : Sort.by(sortBy).descending();

    return ResponseEntity.ok(
        taskRepository.findAll(sort).stream()
            .map(taskMapper::toTaskResponse).toList()
    );
}
```

---
zoom: 0.85
---

# Derived Queries + Pageable

Derived query methods also accept `Pageable`:

```java
public interface TaskRepository extends JpaRepository<TaskModel, Long> {

    // Paginated filter by completed:
    Page<TaskModel> findByCompleted(boolean completed, Pageable pageable);

    // Sorted list of incomplete tasks:
    List<TaskModel> findByCompletedOrderByCreatedAtDesc(boolean completed);

    // Paginated + sorted by method name:
    Page<TaskModel> findByCompleted(boolean completed, Pageable pageable);
}
```

```sql
-- findByCompleted(false, PageRequest.of(0, 5)):
SELECT * FROM tasks WHERE completed = false LIMIT 5 OFFSET 0;
SELECT COUNT(*) FROM tasks WHERE completed = false;
```

---

# Pagination Summary

| Class | Purpose |
|-------|---------|
| `PageRequest.of(page, size)` | Create a `Pageable` for a given page and size |
| `PageRequest.of(page, size, sort)` | Add sorting to a pageable |
| `Sort.by("field").descending()` | Create a sort descriptor |
| `Page<T>` | Return type — wraps List + metadata |
| `page.getContent()` | Extract the `List<T>` from the page |
| `page.getTotalElements()` | Total row count across all pages |

<v-click>

**SQL bridge:** `PageRequest.of(1, 10)` → `LIMIT 10 OFFSET 10`. Spring Data writes the SQL; you write Java.

</v-click>
