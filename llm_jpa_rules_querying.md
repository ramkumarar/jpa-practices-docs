---
description: "LLM rules for writing efficient JPA queries."
alwaysApply: true
---
# JPA Querying Rules

## Rule 1: Avoid N+1 Problems with JOIN FETCH
**Title:** Use `JOIN FETCH` to Eagerly Load Associations
**Description:** To prevent the N+1 query problem, use `JOIN FETCH` in your JPQL query to load parent entities and their required child collections in a single query. Default to `FetchType.LAZY` for all associations.

**Good Example:**
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // Fetches users and their posts in one query
    @Query("SELECT u FROM User u JOIN FETCH u.posts WHERE u.id = :id")
    Optional<User> findByIdWithPosts(@Param("id") Long id);
}

@Entity
public class User {
    // ...
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY) // LAZY is the correct default
    private Set<Post> posts = new HashSet<>();
}
```

**Bad Example:**
```java
import lombok.RequiredArgsConstructor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

// In a service class
@Service
@Transactional
@RequiredArgsConstructor
public class UserService {
    private static final Logger log = LoggerFactory.getLogger(UserService.class);
    private final UserRepository userRepository;

    public void getUserPosts(Long userId) {
        Optional<User> user = userRepository.findById(userId); // First query
        user.ifPresent(u -> {
            // Accessing u.getPosts() triggers N additional queries for N posts
            log.info("Number of posts: {}", u.getPosts().size());
        });
    }
}
```

---

## Rule 2: Use DTO Projections for Read Operations
**Title:** Select Directly into DTOs for Read-Only Views
**Description:** Do not fetch full entities just to map them to DTOs. Use JPQL constructor expressions or interface-based projections to select only the necessary fields directly into a DTO, reducing data transfer and memory usage.

**Good Example (Constructor Expression):**
```java
// DTO with a matching constructor
public class PostDTO {
    private final Long id;
    private final String title;
    public PostDTO(Long id, String title) {
        this.id = id;
        this.title = title;
    }
    // ...
}

@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    @Query("SELECT new com.example.dto.PostDTO(p.id, p.title) FROM Post p WHERE p.userId = :userId")
    List<PostDTO> findPostDTOsByUserId(@Param("userId") Long userId);
}
```

**Good Example (Interface-based Projection):**
```java
public interface PostSummary {
    Long getId();
    String getTitle();
}

@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    List<PostSummary> findByUserId(Long userId);
}
```

**Bad Example:**
```java
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.stream.Collectors;

// In a service class
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;

    public List<PostDTO> getPostDTOs(Long userId) {
        // PROBLEM: Fetches the entire Post entity, which may have many unused fields.
        List<Post> posts = postRepository.findByUserId(userId);
        return posts.stream()
                    .map(post -> new PostDTO(post.getId(), post.getTitle()))
                    .collect(Collectors.toList());
    }
}
```

---

## Rule 3: Separate Queries for Pagination with JOIN FETCH
**Title:** Use Two Queries for Pagination with Joins
**Description:** Do not use `JOIN FETCH` on a collection in the same query as pagination (`Pageable`). This forces in-memory pagination. Instead, use two queries: the first to fetch the paginated IDs of the parent entity, and the second to `JOIN FETCH` the associations for those specific IDs. Consider using `@BatchSize` to optimize the second query.

**Good Example:**
```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    @Query(value = "SELECT p FROM Post p WHERE p.topic = :topic",
           countQuery = "SELECT count(p) FROM Post p WHERE p.topic = :topic")
    Page<Post> findByTopic(@Param("topic") String topic, Pageable pageable);

    @Query("SELECT p FROM Post p JOIN FETCH p.comments WHERE p.id IN :ids")
    List<Post> findWithCommentsByIds(@Param("ids") List<Long> ids);
}

@Entity
public class Post {
    // ...
    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    @BatchSize(size = 20) // Optimizes batch loading
    private Set<Comment> comments = new HashSet<>();
}

import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageImpl;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.stream.Collectors;

// In a service class
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;

    public Page<Post> findPostsWithComments(String topic, Pageable pageable) {
        Page<Post> postPage = postRepository.findByTopic(topic, pageable);
        List<Long> ids = postPage.getContent().stream().map(Post::getId).collect(Collectors.toList());
        List<Post> postsWithComments = postRepository.findWithCommentsByIds(ids);
        return new PageImpl<>(postsWithComments, pageable, postPage.getTotalElements());
    }
}
```

**Bad Example:**
```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    // PROBLEM: This query fetches all posts with their comments into memory
    // before applying pagination, which is highly inefficient.
    @Query("SELECT p FROM Post p JOIN FETCH p.comments WHERE p.topic = :topic")
    Page<Post> findByTopicWithComments(@Param("topic") String topic, Pageable pageable);
}
```

---

## Rule 4: Use `Specification` for Dynamic Queries
**Title:** Build Dynamic Queries with `Specification` or `Criteria` API
**Description:** Avoid building queries using string concatenation, which is unsafe and error-prone. Use the `Specification` interface from Spring Data JPA (or the JPA `Criteria` API) to build type-safe, dynamic queries.

**Good Example:**
```java
public class PostSpecifications {
    public static Specification<Post> hasStatus(String status) {
        return (root, query, cb) -> status == null ? cb.conjunction() : cb.equal(root.get("status"), status);
    }
    public static Specification<Post> createdAfter(LocalDate date) {
        return (root, query, cb) -> date == null ? cb.conjunction() : cb.greaterThanOrEqualTo(root.get("createdAt"), date);
    }
}

import lombok.RequiredArgsConstructor;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.stereotype.Service;

import java.time.LocalDate;
import java.util.List;

// In a service class
@Service
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;

    public List<Post> findPosts(String status, LocalDate date) {
        Specification<Post> spec = Specification.where(PostSpecifications.hasStatus(status))
                                                .and(PostSpecifications.createdAfter(date));
        return postRepository.findAll(spec);
    }
}
```

**Bad Example:**
```java
import jakarta.persistence.EntityManager;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;

import java.time.LocalDate;
import java.util.List;

// In a repository implementation
@Repository
@RequiredArgsConstructor
public class PostRepositoryImpl { // Example implementation
    private final EntityManager entityManager;

    public List<Post> findPosts(String status, LocalDate date) {
        // PROBLEM: Unsafe, hard to read, and prone to SQL injection.
        String qlString = "SELECT p FROM Post p WHERE 1=1";
        if (status != null) {
            qlString += " AND p.status = '" + status + "'";
        }
        if (date != null) {
            qlString += " AND p.createdAt >= '" + date + "'";
        }
        return entityManager.createQuery(qlString, Post.class).getResultList();
    }
}
```

---

## Rule 5: Implement Custom Logic in a Dedicated Repository Implementation
**Title:** Implement Custom Logic in a Dedicated Repository Implementation
**Description:** When you need to implement complex queries that go beyond derived query methods or simple `@Query` annotations (e.g., dynamic queries using the Criteria API, bulk operations using `EntityManager`), the best practice is to create a custom repository implementation. This pattern keeps your data access logic clean, organized, and testable.

**Good Example:**

**1. Define a custom repository interface with your custom method:**
```java
// Custom interface for advanced queries
public interface CustomPostRepository {
    List<Post> findByTitleAndStatus(String title, PostStatus status, Pageable pageable);
}
```

**2. Create an implementation class with the `Impl` suffix:**
Spring Data will automatically discover this implementation. Here you can use the `EntityManager` to build any query you need.
```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jakarta.persistence.criteria.CriteriaBuilder;
import jakarta.persistence.criteria.CriteriaQuery;
import jakarta.persistence.criteria.Predicate;
import jakarta.persistence.criteria.Root;
import org.springframework.data.domain.Pageable;
import java.util.ArrayList;
import java.util.List;

// Implementation of the custom interface
public class CustomPostRepositoryImpl implements CustomPostRepository {

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public List<Post> findByTitleAndStatus(String title, PostStatus status, Pageable pageable) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Post> cq = cb.createQuery(Post.class);
        Root<Post> post = cq.from(Post.class);

        List<Predicate> predicates = new ArrayList<>();
        if (title != null) {
            predicates.add(cb.like(post.get("title"), "%" + title + "%"));
        }
        if (status != null) {
            predicates.add(cb.equal(post.get("status"), status));
        }
        cq.where(predicates.toArray(new Predicate[0]));

        return entityManager.createQuery(cq)
            .setFirstResult((int) pageable.getOffset())
            .setMaxResults(pageable.getPageSize())
            .getResultList();
    }
}
```

**3. Extend your main repository interface with the custom interface:**
This combines the standard `JpaRepository` methods with your custom-defined ones.
```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface PostRepository extends JpaRepository<Post, Long>, CustomPostRepository {
    // You can still have derived query methods here
    // e.g., Optional<Post> findBySlug(String slug);
}
```

**Bad Example:**
Avoids creating a custom repository and instead injects the `EntityManager` into a service to build queries. This mixes data access logic with business logic, making the service harder to test and maintain.
```java
@Service
@RequiredArgsConstructor
public class PostService {
    private final EntityManager entityManager; // Mixing concerns
    private final PostRepository postRepository;

    public List<Post> searchPosts(String title) {
        // BAD: Data access logic is implemented in the service layer.
        // This should be in a repository implementation.
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Post> cq = cb.createQuery(Post.class);
        // ... query logic ...
        return entityManager.createQuery(cq).getResultList();
    }
}
```

---

## Rule 6: Prefer Keyset Pagination Over Offset for Scalable Data Fetching
**Title:** Use Keyset Pagination for Efficient "Infinite Scroll" Style Data Fetching
**Description:** Standard offset-based pagination (used by default in Spring's `Pageable`) becomes very slow on large tables because the database must scan and discard all rows from the beginning. Keyset pagination (also known as the "seek method") avoids this by using a `WHERE` clause to seek directly to the beginning of the next page of results, making it consistently fast. It is the best choice for "load more" or "infinite scroll" features.

**Good Example (Keyset Pagination):**
The query uses the `createdAt` and `id` of the last seen post to fetch the next "page".
```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    @Query("""
        SELECT p FROM Post p
        WHERE p.createdAt < :lastCreatedAt
           OR (p.createdAt = :lastCreatedAt AND p.id < :lastId)
        ORDER BY p.createdAt DESC, p.id DESC
    """)
    List<Post> findNextPage(
        @Param("lastCreatedAt") LocalDateTime lastCreatedAt,
        @Param("lastId") Long lastId,
        Pageable pageable // Use Pageable just for the LIMIT clause
    );
}

// In the service:
public List<Post> getPosts(Post lastSeenPost, int pageSize) {
    LocalDateTime lastCreatedAt = (lastSeenPost != null) ? lastSeenPost.getCreatedAt() : LocalDateTime.now();
    Long lastId = (lastSeenPost != null) ? lastSeenPost.getId() : Long.MAX_VALUE;

    return postRepository.findNextPage(
        lastCreatedAt,
        lastId,
        PageRequest.of(0, pageSize)
    );
}
```

**Bad Example (Offset Pagination):**
This is simple to write but scales very poorly. A request for page 10,000 with a size of 20 would force the database to scan and discard 200,000 rows before returning the 20 results.
```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    // This method is simple but inefficient for deep pages.
    Page<Post> findByStatus(PostStatus status, Pageable pageable);
}

// In the service:
public Page<Post> getPosts(int pageNumber, int pageSize) {
    // PROBLEM: This will be fast for page 0, but very slow for page 10,000.
    return postRepository.findByStatus(
        PostStatus.PUBLISHED,
        PageRequest.of(pageNumber, pageSize, Sort.by("createdAt").descending())
    );
}
```
**Note:** Offset pagination is acceptable for UIs that require jumping to arbitrary page numbers, especially in admin interfaces where performance is less critical. For user-facing infinite scroll, always prefer keyset.

---

## Rule 7: Use `@Transactional(readOnly = true)` for Read Operations
**Title:** Mark Read-Only Operations with `@Transactional(readOnly = true)`
**Description:** Use `@Transactional(readOnly = true)` for all read-only operations. This provides performance benefits by allowing the JPA provider to optimize for read-only scenarios, prevents accidental writes, and provides better connection pool management.

**Good Example:**
```java
@Service
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;

    @Transactional(readOnly = true)
    public List<PostDTO> getPostDTOs(Long userId) {
        return postRepository.findPostDTOsByUserId(userId);
    }

    @Transactional(readOnly = true)
    public Optional<Post> getPostById(Long id) {
        return postRepository.findById(id);
    }

    @Transactional // Write operation - no readOnly flag
    public Post createPost(Post post) {
        return postRepository.save(post);
    }
}
```

**Bad Example:**
```java
@Service
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;

    // PROBLEM: Missing @Transactional(readOnly = true) for read operations
    public List<PostDTO> getPostDTOs(Long userId) {
        return postRepository.findPostDTOsByUserId(userId);
    }

    // PROBLEM: Using readOnly = true for write operations
    @Transactional(readOnly = true)
    public Post createPost(Post post) {
        return postRepository.save(post); // This will fail!
    }
}
```

---

## Rule 8: Avoid `findAll()` Without Constraints
**Title:** Always Use Pagination or Filtering to Prevent Loading Entire Tables
**Description:** Never use `findAll()` or similar methods without constraints in production code. Always use pagination (`Pageable`) or filtering to prevent accidentally loading entire tables into memory, which can cause OutOfMemoryErrors and performance issues.

**Good Example:**
```java
@Service
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;

    @Transactional(readOnly = true)
    public Page<Post> getAllPosts(Pageable pageable) {
        return postRepository.findAll(pageable);
    }

    @Transactional(readOnly = true)
    public List<Post> getRecentPosts() {
        return postRepository.findTop10ByOrderByCreatedAtDesc();
    }

    @Transactional(readOnly = true)
    public List<Post> getPostsByStatus(PostStatus status) {
        return postRepository.findByStatus(status);
    }
}
```

**Bad Example:**
```java
@Service
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;

    @Transactional(readOnly = true)
    public List<Post> getAllPosts() {
        // PROBLEM: This will load ALL posts into memory, potentially millions of records
        return postRepository.findAll();
    }

    @Transactional(readOnly = true)
    public List<Post> getPostsForDropdown() {
        // PROBLEM: Even for dropdowns, this could load too much data
        return postRepository.findAll();
    }
}
```

---

## Rule 9: Use `@EntityGraph` for Complex Fetch Strategies
**Title:** Use `@EntityGraph` for Declarative Fetch Planning
**Description:** When you need to fetch multiple related entities in a single query, use `@EntityGraph` to declaratively specify which associations should be eagerly loaded. This is cleaner than writing multiple `JOIN FETCH` clauses and allows for reusable fetch plans.

**Good Example:**
```java
@Entity
@NamedEntityGraph(
    name = "User.withPostsAndComments",
    attributeNodes = {
        @NamedAttributeNode("posts"),
        @NamedAttributeNode(value = "posts", subgraph = "post-comments")
    },
    subgraphs = @NamedSubgraph(
        name = "post-comments",
        attributeNodes = @NamedAttributeNode("comments")
    )
)
public class User {
    // ... entity fields and relationships
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    @EntityGraph("User.withPostsAndComments")
    Optional<User> findWithPostsAndComments(Long id);
    
    // Alternative inline approach
    @EntityGraph(attributePaths = {"posts", "posts.comments"})
    Optional<User> findWithPostsAndCommentsInline(Long id);
}
```

**Bad Example:**
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // PROBLEM: Complex JOIN FETCH queries become hard to read and maintain
    @Query("""
        SELECT DISTINCT u FROM User u 
        LEFT JOIN FETCH u.posts p 
        LEFT JOIN FETCH p.comments c 
        LEFT JOIN FETCH p.tags t 
        WHERE u.id = :id
    """)
    Optional<User> findWithAllRelations(@Param("id") Long id);
}
```