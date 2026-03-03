---
layout: center
---
# JpaRepository

## The Interface That Replaces ConcurrentHashMap

---

# From Class to Interface

```java
// Before: a class you wrote yourself
@Repository
public class TaskRepository {
    private Map<Long, Task> tasks = new ConcurrentHashMap<>();
    private AtomicLong counter = new AtomicLong(0);

    public List<Task> findAll() { ... }
    public Optional<Task> findById(Long id) { ... }
    public Task save(Task task) { ... }
    public Boolean delete(Long id) { ... }
}

// After: an interface Spring implements for you
public interface TaskRepository extends JpaRepository<Task, Long> {
    // Spring generates the implementation at startup
}
```

<v-click>

No `@Repository` annotation needed — Spring detects JPA repository interfaces automatically.

</v-click>

---

# JpaRepository&lt;Task, Long&gt;

The two type parameters tell Spring what you're storing:

```java
public interface TaskRepository extends JpaRepository<Task, Long> {
    //                                                 ^^^^ ^^^^
    //                                           entity type  ID type
}
```

- `Task` → the `@Entity` class this repository manages
- `Long` → the type of the `@Id` field in `Task`

---

# Built-In Methods

All of these work without writing any code:

```java
// Read
List<Task> tasks = taskRepository.findAll();
Optional<Task> task = taskRepository.findById(1L);
boolean exists = taskRepository.existsById(1L);
long count = taskRepository.count();

// Write
Task saved = taskRepository.save(task);       // INSERT if id==null, UPDATE if id exists
taskRepository.saveAll(listOfTasks);

// Delete
taskRepository.deleteById(1L);
taskRepository.delete(task);
taskRepository.deleteAll();
```

---

# save() — Same Semantics as Before

```java
// Our old TaskRepository.save():
public Task save(Task task) {
    if (task.getId() == null) {
        task.setId(counter.incrementAndGet());  // INSERT
    }
    tasks.put(task.getId(), task);               // UPDATE
    return task;
}

// JpaRepository.save() does the same:
// id == null → INSERT INTO tasks (...) → database assigns id
// id exists → UPDATE tasks SET ... WHERE id = ?
```

<v-click>

The service code doesn't change because `save()` behaves identically.

```java
// TaskServiceImpl.create() — unchanged:
Task task = new Task();
task.setTitle(title);
Task savedTask = taskRepository.save(task);  // works with both repositories
```

</v-click>

---
zoom: 0.85
---

# deleteById() vs our delete()

One difference: our `delete()` returned `Boolean`. JpaRepository's `deleteById()` does not.

```java
// Before:
public Boolean delete(Long id) {
    if (tasks.containsKey(id)) {
        tasks.remove(id);
        return true;
    }
    return false;
}

// JpaRepository:
taskRepository.deleteById(id);
// → throws EmptyResultDataAccessException if id not found
```

<v-click>

We need to update `TaskServiceImpl.delete()` to check existence first:
```java
public boolean delete(Long id) {
    if (!taskRepository.existsById(id)) return false;
    taskRepository.deleteById(id);
    return true;
}
```

</v-click>

---

# Adding Custom Methods

JpaRepository handles standard CRUD. Add your own methods for anything else:

```java
public interface TaskRepository extends JpaRepository<Task, Long> {

    // Spring generates: SELECT * FROM tasks WHERE completed = ?
    List<Task> findByCompleted(boolean completed);

}
```

You declare the method signature. Spring generates the implementation.

This replaces the Java stream filter in the controller:
```java
// Before: loads all tasks, filters in Java
taskRepository.findAll().stream()
    .filter(t -> t.getCompleted().equals(completed))
    .toList();

// After: database does the filtering
taskRepository.findByCompleted(completed);
```

---

# The Interface Hierarchy

```
JpaRepository<T, ID>
  extends PagingAndSortingRepository<T, ID>
    extends CrudRepository<T, ID>
      extends Repository<T, ID>
```

You get all methods from all levels. `JpaRepository` adds JPA-specific features like `flush()` and `saveAndFlush()`.

---
zoom: 0.8
---

# Why the Service Doesn't Change

```java
// TaskServiceImpl.java — identical before and after migration:
@Service
@RequiredArgsConstructor
public class TaskServiceImpl implements TaskService {
    private final TaskRepository taskRepository;  // ← same field name, same methods
    private final TaskMapper taskMapper;

    public List<TaskResponse> findAll() {
        return taskRepository.findAll().stream()
            .map(taskMapper::toTaskResponse).toList();
    }

    public Optional<TaskResponse> findById(Long id) {
        return taskRepository.findById(id).map(taskMapper::toTaskResponse);
    }

    // create(), update(), complete() — all identical
}
```

Both repositories have `findAll()`, `findById()`, `save()`. The service calls the same methods either way.
