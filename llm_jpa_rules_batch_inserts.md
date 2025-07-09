---
description: "LLM rules for efficient JPA batch inserts and bulk operations."
alwaysApply: true
---
# JPA Batch Inserts and Bulk Operations Rules

## Rule 1: Configure JDBC Batching
**Title:** Enable JDBC Batching in Configuration
**Description:** To enable batching, set `spring.jpa.properties.hibernate.jdbc.batch_size` to a value like 50. Also, enable statement ordering with `spring.jpa.properties.hibernate.order_inserts=true` and `spring.jpa.properties.hibernate.order_updates=true`. For MySQL, add `rewriteBatchedStatements=true` to the JDBC URL.

**Good Example (`application.properties`):**
```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.datasource.url=jdbc:mysql://localhost:3306/mydb?rewriteBatchedStatements=true
```

* Tuning batch_size: Use a value between 50–100, depending on workload and database limits. Too large a batch size may cause memory issues.
* JDBC batching limitations: While effective for inserts, it may not optimize updates/deletes as well. For versioned entities (e.g., with @Version), set hibernate.jdbc.batch_versioned_data=true to improve performance.

**Bad Example:**
Absence of `spring.jpa.properties.hibernate.jdbc.batch_size` in `application.properties`.

---

## Rule 2: Use SEQUENCE for IDs
**Title:** Use `GenerationType.SEQUENCE` for Primary Keys in Batch Inserts
**Description:** The `IDENTITY` generation strategy disables insert batching. Always use `SEQUENCE` with an `allocationSize` that matches the `batch_size` to allow Hibernate to batch `INSERT` statements.

**Good Example:**
```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(name = "product_seq", allocationSize = 50)
    private Long id;
    // ...
}
```

**Bad Example:**
```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // Disables batching
    private Long id;
    // ...
}
```

* Lifecycle callbacks: @Modifying does not trigger entity lifecycle callbacks (e.g., @PreUpdate, @PostUpdate) or cascade operations.
* Native queries: For native SQL, use @Query(nativeQuery = true) and ensure proper schema alignment.
* Bulk deletes: Use @Modifying with caution for deletes, as it bypasses JPA’s entity management and may leave associations in an inconsistent state

---

## Rule 3: Use @Modifying for Bulk Operations
**Title:** Use `@Modifying` Queries for Bulk Updates and Deletes
**Description:** To perform bulk `UPDATE` or `DELETE` operations, do not load entities into memory. Instead, define a repository method with a JPQL query annotated with `@Modifying` to execute the operation directly in the database.

**Good Example:**
```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    @Modifying
    @Query("UPDATE Post p SET p.status = :status WHERE p.userId = :userId")
    int updateStatusForUser(@Param("userId") Long userId, @Param("status") String status);
}
```

**Bad Example:**
```java
import lombok.RequiredArgsConstructor;

// In a service class
@Transactional
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;

    public void archivePosts(Long userId) {
        // PROBLEM: Fetches all entities, then updates them one by one.
        List<Post> posts = postRepository.findByUserId(userId);
        for (Post post : posts) {
            post.setStatus("ARCHIVED");
        }
        // This still requires a separate UPDATE for each post.
        // A @Modifying query would issue a single bulk UPDATE statement.
        postRepository.saveAll(posts);
    }
}
```

## Rule 4: Additional recommendations

* Combine batching with `saveAll()`
    * Ensure EntityManager is configured for batching (hibernate.jdbc.batch_size=50) and use repository.saveAll(entities) for batched inserts.
* Avoid mixing ORM and raw SQL:
    * For very large batches, consider JDBC templates or native batch operations for better performance.
* Flush manually in long batches:
    *  For extremely large batch operations, call entityManager.flush() and entityManager.clear() periodically to prevent memory exhaustion.