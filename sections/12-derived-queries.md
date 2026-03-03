---
layout: center
---
# Derived Query Methods

## Spring Generates SQL from Method Names

---

# How Derived Queries Work

Spring reads your method name and generates the SQL query automatically.

```java
public interface TaskRepository extends JpaRepository<TaskModel, Long> {

    List<TaskModel> findByCompleted(boolean completed);

}
```

```sql
-- Spring generates:
SELECT * FROM tasks WHERE completed = ?
```

No SQL written. No `@Query` annotation. Just a method name.

---

# The Naming Convention

```
findBy[Field][Condition](parameter)
  ^^     ^^^     ^^^^^
  │      │       └── e.g. Containing, IgnoreCase, OrderBy
  │      └── field name in Task.java (case-sensitive)
  └── always starts with findBy
```

Examples:

```java
findByCompleted(boolean completed)
// → WHERE completed = ?

findByTitle(String title)
// → WHERE title = ?

findByTitleContaining(String keyword)
// → WHERE title LIKE '%keyword%'
```

---

# Text Matching Keywords

```java
// Exact match
List<TaskModel> findByTitle(String title);
// → WHERE title = ?

// Contains (case-sensitive)
List<TaskModel> findByTitleContaining(String keyword);
// → WHERE title LIKE '%keyword%'

// Contains (case-insensitive)
List<TaskModel> findByTitleContainingIgnoreCase(String keyword);
// → WHERE LOWER(title) LIKE LOWER('%keyword%')

// Starts with
List<TaskModel> findByTitleStartingWith(String prefix);
// → WHERE title LIKE 'prefix%'
```

---

# Sorting with Derived Queries

```java
// Order by createdAt descending
List<TaskModel> findByCompletedOrderByCreatedAtDesc(boolean completed);
// → SELECT * FROM tasks WHERE completed = ? ORDER BY created_at DESC

// Order by title ascending
List<TaskModel> findAllByOrderByTitleAsc();
// → SELECT * FROM tasks ORDER BY title ASC
```

---

# Combining Conditions

```java
// Both conditions must match (AND)
List<TaskModel> findByCompletedAndTitleContaining(boolean completed, String keyword);
// → WHERE completed = ? AND title LIKE '%keyword%'

// Either condition matches (OR)
List<TaskModel> findByCompletedOrTitle(boolean completed, String title);
// → WHERE completed = ? OR title = ?
```

---
zoom: 0.85
---

# findByCompleted — Replacing the Stream Filter

This is the most important one for our Task API:

```java
// TaskRepository.java — add this method:
public interface TaskRepository extends JpaRepository<TaskModel, Long> {
    List<TaskModel> findByCompleted(boolean completed);
}
```

```java
// TaskServiceImpl.java — use it:
public List<TaskResponse> findAll(Boolean completed) {
    List<TaskModel> tasks = (completed != null)
        ? taskRepository.findByCompleted(completed)
        : taskRepository.findAll();
    return tasks.stream().map(taskMapper::toTaskResponse).toList();
}
```

```sql
-- When completed = true:
SELECT * FROM tasks WHERE completed = true

-- When completed = null:
SELECT * FROM tasks
```

---

# Return Types

Derived queries can return different types:

```java
// Returns a list (empty list if nothing found — never null)
List<TaskModel> findByCompleted(boolean completed);

// Returns an Optional (empty if not found)
Optional<TaskModel> findByTitle(String title);

// Returns a single object (throws if not found or multiple found)
Task findOneByTitle(String title);

// Returns a count
long countByCompleted(boolean completed);
// → SELECT COUNT(*) FROM tasks WHERE completed = ?

// Returns whether any match exists
boolean existsByTitle(String title);
// → SELECT COUNT(*) > 0 FROM tasks WHERE title = ?
```

---

# Quick Reference

| Method | Generated SQL |
|--------|--------------|
| `findByCompleted(true)` | `WHERE completed = true` |
| `findByTitleContainingIgnoreCase("jpa")` | `WHERE LOWER(title) LIKE '%jpa%'` |
| `findByCompletedOrderByCreatedAtDesc(false)` | `WHERE completed = false ORDER BY created_at DESC` |
| `countByCompleted(false)` | `SELECT COUNT(*) WHERE completed = false` |
| `existsByTitle("Learn JPA")` | `SELECT COUNT(*) > 0 WHERE title = ?` |

---

# When Derived Queries Aren't Enough

Derived query methods work well for simple filters. They get unwieldy fast when queries become complex.

```java
// Gets messy quickly:
List<TaskModel> findByCompletedAndTitleContainingAndProjectIdAndCreatedAtAfter(...);
```

Next up: `@Query` for custom JPQL and native SQL — and Spring Specifications for fully dynamic filters.
