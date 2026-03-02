---
layout: center
---
# One-to-Many: Project → Task

## @OneToMany and @ManyToOne

---

# The Relationship

One project has many tasks. Each task belongs to exactly one project.

```sql
-- SQL representation:
tasks table has a project_id column (FK → projects.id)

SELECT t.* FROM tasks t
JOIN projects p ON t.project_id = p.id
WHERE p.id = 1;
```

<v-click>

In JPA, we represent this with `@ManyToOne` on the Task side and `@OneToMany` on the Project side.

</v-click>

---

# Step 1: Create Project.java

```java
package com.chetraseng.sunrise_task_flow_api.model;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;
import java.time.LocalDateTime;
import java.util.List;

@Entity
@Table(name = "projects")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Project {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String description;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @OneToMany(mappedBy = "project", cascade = CascadeType.ALL)
    private List<Task> tasks;
}
```

---

# Step 2: Update Task.java

Add the `@ManyToOne` relationship:

```java
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
    private LocalDateTime createdAt;

    @ManyToOne
    @JoinColumn(name = "project_id")    // ← the FK column in tasks table
    private Project project;            // ← replaces the old Long projectId
}
```

---

# Annotation Meanings

```java
// On Task — "many tasks belong to one project":
@ManyToOne
@JoinColumn(name = "project_id")
private Project project;
// SQL: project_id BIGINT REFERENCES projects(id)

// On Project — "one project has many tasks":
@OneToMany(mappedBy = "project", cascade = CascadeType.ALL)
private List<Task> tasks;
// SQL: no new column — the FK is already in tasks.project_id
// mappedBy = "project" means "look at Task.project for the FK"
```

---

# CascadeType.ALL

```java
@OneToMany(mappedBy = "project", cascade = CascadeType.ALL)
private List<Task> tasks;
```

`cascade = CascadeType.ALL` means: when you do something to the project, cascade it to its tasks.

| Operation on Project | Cascade Effect |
|----------------------|----------------|
| `save(project)` | saves all tasks too |
| `delete(project)` | deletes all tasks too |

<v-click>

Use `CascadeType.ALL` when child entities shouldn't exist without their parent.
Tasks without a project = orphaned data.

</v-click>

---

# ProjectRepository

```java
package com.chetraseng.sunrise_task_flow_api.repository;

import com.chetraseng.sunrise_task_flow_api.model.Project;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProjectRepository extends JpaRepository<Project, Long> {
    // findAll, findById, save, deleteById — all available
}
```

Same pattern as `TaskRepository`. One line. Spring generates the implementation.

---

# Updated TaskRepository

```java
public interface TaskRepository extends JpaRepository<Task, Long> {

    List<Task> findByCompleted(boolean completed);

    // Get all tasks for a project
    List<Task> findByProject(Project project);

    // Or by project ID directly
    List<Task> findByProjectId(Long projectId);

}
```

```sql
-- findByProjectId(1L):
SELECT * FROM tasks WHERE project_id = 1
```

---

# Creating a Task With a Project

```java
// In TaskServiceImpl:
public TaskResponse create(String title, String description, Long projectId) {
    Project project = projectRepository.findById(projectId)
        .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
            "Project not found: " + projectId));

    Task task = Task.builder()
        .title(title)
        .description(description)
        .project(project)          // ← set the relationship object
        .completed(false)
        .build();

    return taskMapper.toTaskResponse(taskRepository.save(task));
}
```

---

# Generated SQL

```bash
# POST /api/tasks with {"title":"New task","description":"...","projectId":1}
```

```sql
-- JPA runs:
SELECT * FROM projects WHERE id = 1;   -- validate project exists

INSERT INTO tasks (title, description, completed, created_at, project_id)
VALUES ('New task', '...', false, NOW(), 1);
```

The `project_id` column is populated automatically from the `Task.project` reference.

---

# Avoid Infinite JSON Loops

`Task` has `Project project`, and `Project` has `List<Task> tasks`.
JSON serialization will loop forever without `@JsonIgnore`:

```java
// In Project.java:
@OneToMany(mappedBy = "project", cascade = CascadeType.ALL)
@JsonIgnore
private List<Task> tasks;
```

Or use `@JsonManagedReference` / `@JsonBackReference` for bi-directional serialization.

For this course, `@JsonIgnore` on the `tasks` side is simplest. Expose task data through the `TaskResponse` DTO.
