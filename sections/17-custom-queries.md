---
layout: center
---
# Custom Queries

## @Query, JPQL, and Native SQL

---

# Three Ways to Query

| Approach | When to use |
|----------|------------|
| Derived method (`findByCompleted`) | Simple, single-field filters |
| `@Query` with JPQL | Complex logic, joins, expressions |
| `@Query` with native SQL | DB-specific features, raw SQL control |

---

# What is JPQL?

**JPQL** (Java Persistence Query Language) looks like SQL but operates on **Java entities**, not tables.

```java
// SQL:   SELECT * FROM tasks WHERE completed = true
// JPQL:  SELECT t FROM Task t WHERE t.completed = true
//                   ^^^^                ^^^^
//              Java class name      Java field name
```

JPQL is database-agnostic — Hibernate translates it to the right SQL dialect for PostgreSQL, H2, MySQL, etc.

---

# @Query — Basic JPQL

```java
public interface TaskRepository extends JpaRepository<Task, Long> {

    @Query("SELECT t FROM Task t WHERE t.completed = false ORDER BY t.createdAt DESC")
    List<Task> findAllIncompleteOrderedByDate();

}
```

<v-click>

```sql
-- Hibernate generates:
SELECT t.id, t.title, t.description, t.completed, t.created_at
FROM tasks t
WHERE t.completed = false
ORDER BY t.created_at DESC
```

</v-click>

---

# @Param — Named Parameters

```java
@Query("SELECT t FROM Task t WHERE t.title LIKE %:keyword% AND t.completed = :completed")
List<Task> search(@Param("keyword") String keyword,
                  @Param("completed") boolean completed);
```

- `:keyword` is a named parameter — matched by `@Param("keyword")`
- `%:keyword%` — wrapping with `%` works for contains matching in JPQL

<v-click>

```java
// Usage:
taskRepository.search("spring", false);
// → WHERE title LIKE '%spring%' AND completed = false
```

</v-click>

---

# @Query with JOIN

JPQL can join across entities using the Java field name, not the column name:

```java
@Query("SELECT t FROM Task t JOIN t.project p WHERE p.name = :projectName")
List<Task> findByProjectName(@Param("projectName") String projectName);
```

```sql
-- Hibernate generates:
SELECT t.* FROM tasks t
JOIN projects p ON t.project_id = p.id
WHERE p.name = ?
```

No raw SQL needed — JPA resolves the join through the `@ManyToOne` relationship.

---

# Native Queries

When you need raw SQL — PostgreSQL-specific functions, complex subqueries, or performance tuning:

```java
@Query(
    value = "SELECT * FROM tasks WHERE title ILIKE %:keyword%",
    nativeQuery = true
)
List<Task> searchNative(@Param("keyword") String keyword);
```

- `nativeQuery = true` → runs the SQL directly without JPQL translation
- `ILIKE` is PostgreSQL-specific case-insensitive LIKE — not available in JPQL
- Returns mapped to `Task` entity automatically

---

# Native Query — Projection

For native queries that don't return a full entity, use a projection interface:

```java
public interface TaskSummary {
    Long getId();
    String getTitle();
    Boolean getCompleted();
}

@Query(value = "SELECT id, title, completed FROM tasks WHERE project_id = :pid",
       nativeQuery = true)
List<TaskSummary> findSummariesByProject(@Param("pid") Long projectId);
```

---

# @Modifying — UPDATE and DELETE Queries

`@Query` can also run `UPDATE` and `DELETE`. Add `@Modifying` + `@Transactional`:

```java
@Modifying
@Transactional
@Query("UPDATE Task t SET t.completed = true WHERE t.id = :id")
int markComplete(@Param("id") Long id);
```

```java
@Modifying
@Transactional
@Query("DELETE FROM Task t WHERE t.completed = true")
int deleteAllCompleted();
```

<v-click>

Returns `int` — the number of rows affected.

`@Modifying` is required for any query that changes data. `@Transactional` ensures it runs inside a transaction.

</v-click>

---

# Derived vs @Query vs Native — When to Use Each

```java
// ✅ Derived — simple, readable, no SQL needed
List<Task> findByCompleted(boolean completed);

// ✅ @Query JPQL — complex logic, stays DB-agnostic
@Query("SELECT t FROM Task t WHERE t.title LIKE %:kw% AND t.completed = false")
List<Task> search(@Param("kw") String keyword);

// ✅ Native — DB-specific features, raw performance control
@Query(value = "SELECT * FROM tasks WHERE title ILIKE %:kw%", nativeQuery = true)
List<Task> searchNative(@Param("kw") String keyword);
```

Start with derived methods. Move to `@Query` when the method name becomes unreadable. Use native only when JPQL can't express what you need.
