---
description: "LLM rules for other common JPA pitfalls and best practices."
alwaysApply: true
---
# Other JPA Rules

## Rule 1: Use `@Enumerated(EnumType.STRING)` for Enums
**Title:** Always Persist Enums as Strings
**Description:** The default JPA behavior is to persist enums using their ordinal (integer) value, which is brittle and dangerous. If you reorder the enum constants, you will corrupt your data. Always use `@Enumerated(EnumType.STRING)` to store the enum's name, which is safe and readable.

**Good Example:**
```java
@Entity
public class Task {
    public enum TaskStatus { PENDING, COMPLETE }

    @Enumerated(EnumType.STRING)
    @Column(length = 20)
    private TaskStatus status;
}
```

**Bad Example:**
```java
@Entity
public class Task {
    // PROBLEM: Uses the default OrdinalType.
    @Enumerated
    private TaskStatus status;
}
```

---

## Rule 2: Use `@DynamicUpdate` for Wide Entities
**Title:** Use `@DynamicUpdate` for Entities with Many Columns
**Description:** By default, Hibernate updates all columns in an entity, even if only one has changed. For entities with many columns or LOBs, this is inefficient. Use the `@DynamicUpdate` annotation to tell Hibernate to generate `UPDATE` statements that only include the columns that have actually changed.

**Good Example:**
```java
@Entity
@DynamicUpdate
public class UserProfile {
    @Id private Long id;
    private String username;
    private String bio;
    private Instant lastLogin;
    // ... many other fields
}
```

**Bad Example:**
An entity with many columns that is not annotated with `@DynamicUpdate`.

---

## Rule 3: Disable Open Session in View
**Title:** Disable the Open Session in View Anti-Pattern
**Description:** The Open Session in View (OSIV) pattern holds database connections open for the entire duration of a web request, which leads to connection pool exhaustion under load. Always disable it and fetch all necessary data within a short service-layer transaction, passing DTOs to the view layer.

**Good Example (`application.properties`):**
```properties
spring.jpa.open-in-view=false
```

**Bad Example:**
Leaving `spring.jpa.open-in-view` as its default value (`true`).

---

## Rule 4: Avoid `CascadeType.REMOVE`
**Title:** Do Not Use `CascadeType.REMOVE` on Collections
**Description:** Using `CascadeType.REMOVE` on a collection is dangerous. It can cause massive, unintended deletions and performs very poorly because it loads the entire collection into memory to delete children one by one. Instead, handle deletions explicitly with bulk `@Modifying` queries in your service layer.

**Good Example:**
```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    @Modifying
    @Query("DELETE FROM Post p WHERE p.author.id = :authorId")
    void deleteByAuthorId(@Param("authorId") Long authorId);
}

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

// In AuthorService
@Service
@Transactional
@RequiredArgsConstructor
public class AuthorService {
    private final AuthorRepository authorRepository;
    private final PostRepository postRepository;

    public void deleteAuthor(Long authorId) {
        postRepository.deleteByAuthorId(authorId); // Explicitly delete children first
        authorRepository.deleteById(authorId);
    }
}
```

**Bad Example:**
```java
@Entity
public class Author {
    // ...
    // DANGEROUS: This can cause massive, slow deletions.
    @OneToMany(mappedBy = "author", cascade = CascadeType.REMOVE)
    private List<Post> posts;
}
```
