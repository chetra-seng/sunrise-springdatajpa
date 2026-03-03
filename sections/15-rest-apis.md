---
layout: center
---
# One-to-Many: Task → Comment

## Nested Relationships with JPA

---

# The Relationship

Each task can have many comments. Each comment belongs to exactly one task.

```sql
CREATE TABLE comments (
    id          BIGSERIAL    PRIMARY KEY,
    content     TEXT         NOT NULL,
    created_at  TIMESTAMP    NOT NULL DEFAULT NOW(),
    task_id     BIGINT       REFERENCES tasks(id)
);
```

```sql
-- Get all comments for task 3:
SELECT * FROM comments WHERE task_id = 3 ORDER BY created_at ASC;
```

---
zoom: 0.85
---

# Comment.java

```java
package com.chetraseng.sunrise_task_flow_api.model;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;
import java.time.LocalDateTime;

@Entity
@Table(name = "comments")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Comment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String content;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @ManyToOne
    @JoinColumn(name = "task_id")
    private Task task;
}
```

---
zoom: 0.85
---

# Update Task.java — Add Comments

```java
@Entity
@Table(name = "tasks")
@Data @NoArgsConstructor @AllArgsConstructor @Builder
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
    @JoinColumn(name = "project_id")
    private Project project;

    @OneToMany(mappedBy = "task", cascade = CascadeType.ALL)
    @JsonIgnore
    private List<Comment> comments;    // ← new
}
```

---

# CommentRepository

```java
package com.chetraseng.sunrise_task_flow_api.repository;

import com.chetraseng.sunrise_task_flow_api.model.Comment;
import com.chetraseng.sunrise_task_flow_api.model.Task;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface CommentRepository extends JpaRepository<Comment, Long> {

    List<Comment> findByTask(Task task);
    List<Comment> findByTaskId(Long taskId);

}
```

```sql
-- findByTaskId(3L):
SELECT * FROM comments WHERE task_id = 3
```

---

# Comment DTOs

```java
// Input: client sends only content
@Data @NoArgsConstructor @AllArgsConstructor
public class CommentRequest {
    private String content;
}

// Output: system returns id, content, createdAt
@Data @NoArgsConstructor @AllArgsConstructor
public class CommentResponse {
    private Long id;
    private String content;
    private LocalDateTime createdAt;
    private Long taskId;
}
```

---

# CommentMapper

```java
package com.chetraseng.sunrise_task_flow_api.mapper;

import com.chetraseng.sunrise_task_flow_api.dto.CommentResponse;
import com.chetraseng.sunrise_task_flow_api.model.Comment;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.MappingConstants;

@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface CommentMapper {

    @Mapping(target = "taskId", source = "task.id")
    CommentResponse toCommentResponse(Comment comment);

}
```

`task.id` → the ID from the nested `Task` object. MapStruct traverses the relationship.

---
zoom: 0.8
---

# CommentController — Nested Routes

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/tasks/{taskId}/comments")
public class CommentController {

    private final CommentRepository commentRepository;
    private final TaskRepository taskRepository;
    private final CommentMapper commentMapper;

    @GetMapping
    public ResponseEntity<List<CommentResponse>> getComments(@PathVariable Long taskId) {
        if (!taskRepository.existsById(taskId)) {
            return ResponseEntity.notFound().build();
        }
        List<CommentResponse> comments = commentRepository.findByTaskId(taskId)
            .stream().map(commentMapper::toCommentResponse).toList();
        return ResponseEntity.ok(comments);
    }
}
```

---
zoom: 0.8
---

# Adding a Comment

```java
@PostMapping
public ResponseEntity<CommentResponse> addComment(
        @PathVariable Long taskId,
        @RequestBody CommentRequest request) {

    Task task = taskRepository.findById(taskId)
        .orElse(null);
    if (task == null) return ResponseEntity.notFound().build();

    Comment comment = Comment.builder()
        .content(request.getContent())
        .task(task)          // ← set the FK relationship
        .build();

    Comment saved = commentRepository.save(comment);
    return ResponseEntity.status(HttpStatus.CREATED)
        .body(commentMapper.toCommentResponse(saved));
}
```

```sql
-- JPA runs:
INSERT INTO comments (content, created_at, task_id) VALUES ('...', NOW(), 3);
```

---

# Pattern Recap: Three Levels Deep

```
Project  ──(has many)──▶  Task  ──(has many)──▶  Comment
  @OneToMany               @OneToMany               @ManyToOne
  mappedBy="project"       mappedBy="task"          @JoinColumn(task_id)
                 @ManyToOne
                 @JoinColumn(project_id)
```

Every `@OneToMany` has a matching `@ManyToOne` on the other side. The FK column always lives on the "many" side.
