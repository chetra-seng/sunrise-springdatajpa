---
layout: center
---
# JpaRepository

## The Interface That Replaces ConcurrentHashMap

---

# JpaRepository&lt;TaskModel, Long&gt;

The two type parameters tell Spring what you're storing:

```java
public interface TaskRepository extends JpaRepository<TaskModel, Long> {
    //                                                 ^^^^ ^^^^
    //                                           entity type  ID type
}
```

- `TaskModel` → the `@Entity` class this repository manages
- `Long` → the type of the `@Id` field in `TaskModel`

---

# Built-In Methods

All of these work without writing any code:

```java
// Read
List<TaskModel> tasks = taskRepository.findAll();
Optional<TaskModel> task = taskRepository.findById(1L);
boolean exists = taskRepository.existsById(1L);
long count = taskRepository.count();

// Write
TaskModel saved = taskRepository.save(task);       // INSERT if id==null, UPDATE if id exists
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
public TaskModel save(TaskModel task) {
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
TaskModel task = new TaskModel();
task.setTitle(title);
TaskModel savedTask = taskRepository.save(task);  // works with both repositories
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
public interface TaskRepository extends JpaRepository<TaskModel, Long> {

    // Spring generates: SELECT * FROM tasks WHERE completed = ?
    List<TaskModel> findByCompleted(boolean completed);

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
