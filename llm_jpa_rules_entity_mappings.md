---
description: "LLM rules for best practices in mapping JPA entity relationships."
alwaysApply: true
---
# JPA Entity Relationship and Mapping Rules

## Rule 1: Always Default to `FetchType.LAZY` for All Associations
**Title:** Make `FetchType.LAZY` Your Default for All Entity Associations
**Description:** Eager fetching (`FetchType.EAGER`) is a major source of performance problems, often leading to unintentional N+1 queries or fetching excessive data. Always define `@...ToMany` associations as `LAZY`. For `@...ToOne` associations, `LAZY` is also the strongly recommended default. If you need an association to be loaded, explicitly fetch it in a query using `JOIN FETCH`.

**Good Example:**
```java
@Entity
public class Post {
    // ...
    @ManyToOne(fetch = FetchType.LAZY) // LAZY is the best default
    @JoinColumn(name = "author_id")
    private Author author;
}

@Entity
public class Author {
    // ...
    // @OneToMany is LAZY by default, which is correct.
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Post> posts = new ArrayList<>();
}
```

**Bad Example:**
```java
@Entity
public class Post {
    // ...
    // PROBLEM: EAGER fetching will always execute a join, even when you only need the post.
    // This becomes a major issue with multiple EAGER associations.
    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "author_id")
    private Author author;
}
```

---

## Rule 2: Manage Bidirectional Associations Correctly
**Title:** Use Helper Methods to Keep Bidirectional Associations in Sync
**Description:** In a bidirectional `@OneToMany` <-> `@ManyToOne` relationship, you are responsible for keeping both sides of the association in sync. Failure to do so will lead to an inconsistent object model and potential `NullPointerException`s. The best practice is to add helper methods on the "one" side of the relationship to manage the "many" side.

**Good Example (with helper methods):**
```java
@Entity
public class Author {
    // ...
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Post> posts = new ArrayList<>();

    // Helper method to add a post
    public void addPost(Post post) {
        posts.add(post);
        post.setAuthor(this); // CRITICAL: Set the other side of the association
    }

    // Helper method to remove a post
    public void removePost(Post post) {
        posts.remove(post);
        post.setAuthor(null); // CRITICAL: Unset the other side
    }
}

// In a service:
@Transactional
public void createPost(Long authorId, String postTitle) {
    Author author = authorRepository.findById(authorId).orElseThrow();
    Post newPost = new Post(postTitle);
    author.addPost(newPost); // Use the helper method to ensure consistency
    // No need to save the post explicitly if cascade is configured
}
```

**Bad Example (manual and error-prone):**
```java
// In a service:
@Transactional
public void createPost(Long authorId, String postTitle) {
    Author author = authorRepository.findById(authorId).orElseThrow();
    Post newPost = new Post(postTitle);

    // PROBLEM: Manually setting both sides is easy to forget.
    // If a developer forgets newPost.setAuthor(author), the foreign key will be null.
    author.getPosts().add(newPost);
    newPost.setAuthor(author);
}
```

---

## Rule 3: Avoid `@ManyToMany` in Favor of a Join Table Entity
**Title:** Model Many-to-Many Relationships with a Dedicated Join Entity
**Description:** While JPA offers `@ManyToMany`, it hides the join table and makes it difficult to add columns to the relationship (e.g., a `createdAt` timestamp). It also has confusing cascading behavior. The robust and flexible approach is to map the join table as a dedicated entity, effectively converting one `@ManyToMany` relationship into two `@OneToMany` relationships.

**Good Example (using a dedicated join entity):**
```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<PostTag> tags = new ArrayList<>();
    // ...
}

@Entity
public class Tag {
    @OneToMany(mappedBy = "tag", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<PostTag> posts = new ArrayList<>();
    // ...
}

// The dedicated join entity
@Entity
public class PostTag {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private Post post;

    @ManyToOne(fetch = FetchType.LAZY)
    private Tag tag;

    private LocalDateTime createdAt = LocalDateTime.now(); // Extra column on the relationship
}
```

**Bad Example (using `@ManyToMany`):**
```java
@Entity
public class Post {
    // PROBLEM: Difficult to add columns to the join table (e.g., when the tag was added).
    // Cascading can be unpredictable.
    @ManyToMany(cascade = { CascadeType.PERSIST, CascadeType.MERGE })
    @JoinTable(name = "post_tag",
        joinColumns = @JoinColumn(name = "post_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private Set<Tag> tags = new HashSet<>();
}
```

---

## Rule 4: Use `CascadeType` with Extreme Caution
**Title:** Use `CascadeType` Deliberately and Avoid `CascadeType.ALL`
**Description:** `CascadeType.ALL` is convenient but dangerous, as it includes the risky `CascadeType.REMOVE`. Only cascade operations from parent entities to child entities. The most common and safest cascade types are `PERSIST`, `MERGE`, and `REFRESH`. Use `orphanRemoval=true` for automatically deleting child entities when they are removed from a parent's collection, which is often safer and more explicit than `CascadeType.REMOVE`.

**Good Example (specific and safe cascading):**
```java
@Entity
public class Author {
    // ...
    // This is a common and safe setup for a parent-child relationship.
    // New posts added to an author will be persisted.
    // When a post is removed from the list, it will be deleted from the DB.
    @OneToMany(
        mappedBy = "author",
        cascade = { CascadeType.PERSIST, CascadeType.MERGE }, // Only cascade saves and updates
        orphanRemoval = true // Safer way to handle deletions
    )
    private List<Post> posts = new ArrayList<>();
}
```

**Bad Example (overly broad and risky):**
```java
@Entity
public class Author {
    // ...
    // PROBLEM: CascadeType.ALL includes CascadeType.REMOVE. If you delete an Author,
    // it will trigger a cascade delete of all their posts. This might be desired,
    // but it's often safer to handle such deletions explicitly in a service layer.
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL)
    private List<Post> posts = new ArrayList<>();
}
```
