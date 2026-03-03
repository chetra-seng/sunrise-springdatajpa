---
layout: center
---
# Spring Specifications

## Dynamic Queries Without Writing SQL

---

# The Problem With Derived Queries

What if the user can filter by any combination of fields?

```java
GET /api/tasks?completed=true
GET /api/tasks?completed=true&keyword=spring
GET /api/tasks?completed=true&keyword=spring&projectId=1
GET /api/tasks?projectId=1
```

You'd need a derived method for every combination:

```java
findByCompleted(boolean completed);
findByCompletedAndTitleContaining(boolean completed, String keyword);
findByCompletedAndTitleContainingAndProjectId(boolean completed, String keyword, Long projectId);
// ... this explodes fast
```

---

# What is a Specification?

A **Specification** is a reusable, composable piece of a `WHERE` clause.

```java
Specification<Task> isComplete   = (root, query, cb) -> cb.isTrue(root.get("completed"));
Specification<Task> hasKeyword   = (root, query, cb) -> cb.like(root.get("title"), "%spring%");
Specification<Task> inProject    = (root, query, cb) -> cb.equal(root.get("project").get("id"), 1L);

// Combine at runtime:
taskRepository.findAll(isComplete.and(hasKeyword).and(inProject));
```

```sql
-- Generates:
WHERE completed = true
  AND title LIKE '%spring%'
  AND project_id = 1
```

---

# Enable Specifications on the Repository

Add `JpaSpecificationExecutor` to `TaskRepository`:

```java
public interface TaskRepository
        extends JpaRepository<Task, Long>,
                JpaSpecificationExecutor<Task> {    // ← add this

    List<Task> findByCompleted(boolean completed);
}
```

This gives you `findAll(Specification<Task> spec)` for free.

---
zoom: 0.75
---

# Writing Specifications

A `Specification<T>` is a functional interface with one method:

```java
@FunctionalInterface
public interface Specification<T> {
    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);
    //                    ^^^^ entity  ^^^^^ query control    ^^^^ builds conditions
}
```

Put your specifications in a static factory class:

```java
public class TaskSpecs {

    public static Specification<Task> isCompleted(boolean completed) {
        return (root, query, cb) -> cb.equal(root.get("completed"), completed);
    }

    public static Specification<Task> titleContains(String keyword) {
        return (root, query, cb) ->
            cb.like(cb.lower(root.get("title")), "%" + keyword.toLowerCase() + "%");
    }

    public static Specification<Task> inProject(Long projectId) {
        return (root, query, cb) -> cb.equal(root.get("project").get("id"), projectId);
    }
}
```

---

# Composing Specifications

Combine with `.and()`, `.or()`, `.not()`:

```java
import static com.chetraseng.sunrise_task_flow_api.spec.TaskSpecs.*;

// All three filters applied:
Specification<Task> spec = isCompleted(true)
    .and(titleContains("spring"))
    .and(inProject(1L));

List<Task> results = taskRepository.findAll(spec);
```

```sql
WHERE completed = true
  AND LOWER(title) LIKE '%spring%'
  AND project_id = 1
```

---

# Using Specifications in the Service

Build the specification dynamically based on which params are present:

```java
public List<TaskResponse> search(Boolean completed, String keyword, Long projectId) {
    Specification<Task> spec = Specification.where(null);

    if (completed != null) spec = spec.and(isCompleted(completed));
    if (keyword   != null) spec = spec.and(titleContains(keyword));
    if (projectId != null) spec = spec.and(inProject(projectId));

    return taskRepository.findAll(spec).stream()
        .map(taskMapper::toTaskResponse)
        .toList();
}
```

Each filter is only applied when the parameter is actually provided. No `null` checks scattered across a long method chain.

---

# Handling Optional Specs Cleanly

`Specification.where(null)` is a no-op — it matches everything.

```java
Specification<Task> spec = Specification.where(null);
// → no WHERE clause yet

spec = spec.and(isCompleted(true));
// → WHERE completed = true

spec = spec.and(titleContains("spring"));
// → WHERE completed = true AND title LIKE '%spring%'
```

Start with `where(null)` and `.and()` onto it safely — each step is optional.

---

# Derived vs @Query vs Specification

| | Derived method | @Query | Specification |
|-|---------------|--------|---------------|
| SQL written by | Spring | You (JPQL/SQL) | You (Java API) |
| Dynamic filters | ❌ fixed at compile time | ❌ fixed at compile time | ✅ composed at runtime |
| Readable | ✅ for simple cases | ✅ | ✅ with factory methods |
| Best for | Simple fixed filters | Complex fixed queries | Dynamic optional filters |

Use **Specifications** when the combination of filters isn't known until runtime — like a search endpoint with many optional parameters.
