---
description: "LLM rules for using native SQL queries with JPA and PostgreSQL."
alwaysApply: true
---
# JPA Native Query Rules for PostgreSQL

## Rule 1: Prefer JPQL; Use Native Queries for Specific PostgreSQL Features
**Title:** Default to JPQL for Portability and Use Native Queries for Database-Specific Power
**Description:** Your default choice for writing queries should always be JPQL (or the Criteria API). It is database-agnostic, works directly with your object model, and is generally safer. However, you should switch to a native query when you need to leverage powerful, database-specific features that JPQL cannot support. This is a deliberate trade-off, sacrificing portability for power.

### When to use JPQL (The Default Choice):
*   For all standard CRUD operations and business logic that can be expressed in terms of entities and their relationships.
*   When you want your data access layer to remain portable across different database vendors.
*   When you want to leverage JPA's object-oriented query language and automatic entity state management.

**Good JPQL Example (Portable and Object-Oriented):**
```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    @Query("""
        SELECT p FROM Post p
        WHERE p.status = :status
        AND p.createdAt >= :date
    """)
    List<Post> findByStatusAndDate(PostStatus status, LocalDateTime date);
}
```

### When to use a Native Query (The Exception):

* **Leveraging Advanced PostgreSQL Features:** This includes querying jsonb columns, using full-text search functions (to_tsvector), leveraging geospatial features from PostGIS, executing recursive queries or Common Table Expressions (CTEs), and utilizing advanced window functions. These are functionalities not directly supported by JPQL.
* **Performance Optimization for Complex Queries:** When attempting to achieve complex logic with JPQL results in poor performance, a carefully crafted native query might offer advantages. Always benchmark thoroughly before making this decision.
* **Fine-Grained Query Tuning (Rare):** In specific cases where you need to provide hints to the PostgreSQL query planner that Hibernate's SQL generation doesn’t support.

**Important Note on Native Queries:**  Always use parameterized queries (e.g., ? placeholders) in your native SQL to prevent SQL injection vulnerabilities.  Never concatenate user-provided data directly into the query string.

**Good Native Query Example (Leveraging `jsonb`):**
This query finds posts where a `jsonb` attributes column contains a specific property. This is impossible with standard JPQL.
```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    @Query(
        value = "SELECT * FROM post WHERE attributes ->> 'author_rank' = :rank",
        nativeQuery = true
    )
    List<Post> findByJsonAttribute(@Param("rank") String rank);
}
```

---

## Rule 2: Execute Native Queries Safely with Result Set Mapping
**Title:** Always Use Parameter Binding and Map Native Query Results to DTOs
**Description:** When you write a native query, you lose some of JPA's automatic safety nets. It is your responsibility to prevent SQL injection and to handle the raw results correctly. The best practice is to always use named parameters and to map the results directly into a DTO using `@SqlResultSetMapping`.

**Good Example (Safe, Clean, and Mapped):**

**1. Define the DTO:** This is the target for your query results.
```java
public class PostSummaryDTO {
    private final Long id;
    private final String title;
    private final String authorName;

    public PostSummaryDTO(Long id, String title, String authorName) {
        this.id = id;
        this.title = title;
        this.authorName = authorName;
    }
    // Getters...
}
```

**2. Define the `@SqlResultSetMapping`:** Place this on an entity (e.g., the main entity of the repository). It declaratively defines how to map columns to DTO constructor arguments.
```java
@Entity
@SqlResultSetMapping(
    name = "PostSummaryMapping",
    classes = {@ConstructorResult(
        targetClass = PostSummaryDTO.class,
        columns = {
            @ColumnResult(name = "id", type = Long.class),
            @ColumnResult(name = "title"),
            @ColumnResult(name = "author_name")
        }
    )}
)
public class Post {
    // ... entity fields ...
}
```

**3. Write the Native Query in the Repository:** Use the `nativeQuery=true` flag and reference the mapping by name. **Crucially, use named parameters (`:status`) to prevent SQL injection.**
```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    @Query(
        value = """
            SELECT
                p.id AS id,
                p.title AS title,
                a.name AS author_name
            FROM post p
            JOIN author a ON p.author_id = a.id
            WHERE p.status = :status
        """,
        nativeQuery = true,
        resultSetMapping = "PostSummaryMapping"
    )
    List<PostSummaryDTO> findPostSummariesByStatus(@Param("status") String status);
}
```

**Alternative Approach (Spring Data JPA 2.1+):**
If you prefer to avoid `@SqlResultSetMapping`, you can use interface-based projections or class-based projections directly:
```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    @Query(
        value = """
            SELECT
                p.id AS id,
                p.title AS title,
                a.name AS authorName
            FROM post p
            JOIN author a ON p.author_id = a.id
            WHERE p.status = :status
        """,
        nativeQuery = true
    )
    List<PostSummaryDTO> findPostSummariesByStatus(@Param("status") String status);
}
```

**Bad Example (Unsafe and Unwieldy):**
```java
// In a service or repository
public List<Object[]> unsafeSearch(String status) {
    // HUGE PROBLEM: String concatenation is vulnerable to SQL injection.
    String sql = "SELECT p.id, p.title FROM post p WHERE p.status = '" + status + "'";

    Query query = entityManager.createNativeQuery(sql);

    // ANOTHER PROBLEM: Returns List<Object[]>, which is not type-safe and
    // requires manual, error-prone casting and indexing in the service layer.
    return query.getResultList();
}
```

---

## Rule 3: Use `FOR UPDATE SKIP LOCKED` for Concurrent Work Queues
**Title:** Implement High-Throughput Work Queues with `SELECT FOR UPDATE SKIP LOCKED`
**Description:** When you have multiple application instances (workers) competing to process jobs from a table, a standard `SELECT ... FOR UPDATE` will cause workers to wait for each other, creating a bottleneck. PostgreSQL's `SKIP LOCKED` clause is the solution. It allows a worker to lock the rows it selects while other workers can immediately skip those locked rows and select the next available ones. This pattern is essential for building scalable, concurrent job processing systems.

**Good Example (Concurrent-Safe Task Polling):**
This native query safely fetches a batch of pending tasks, locks them for the current transaction, and allows other workers to fetch a different batch without blocking.
```java
@Repository
public interface TaskRepository extends JpaRepository<Task, Long> {
    @Query(
        value = """
            SELECT * FROM task
            WHERE status = 'PENDING'
            ORDER BY created_at ASC
            LIMIT :limit
            FOR UPDATE SKIP LOCKED
        """,
        nativeQuery = true
    )
    List<Task> findAndLockNextPendingTasks(@Param("limit") int limit);
}

// In a transactional service:
@Service
@RequiredArgsConstructor
public class TaskProcessorService {
    private final TaskRepository taskRepository;

    @Transactional
    public void processTasks() {
        // 1. Fetch and lock a batch of tasks. Other workers can run this simultaneously.
        List<Task> tasks = taskRepository.findAndLockNextPendingTasks(10);

        for (Task task : tasks) {
            // 2. Process the task
            task.setStatus("PROCESSING");
            // ... do work ...
            task.setStatus("COMPLETE");
        }
        // 3. The transaction commits, releasing the locks.
        // Note: Locks are also released if the transaction rolls back.
    }
}
```

**Bad Example (Blocking Lock):**
Using a standard pessimistic lock would cause all but one worker to wait, defeating the purpose of having multiple workers.
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT t FROM Task t WHERE t.status = 'PENDING' ORDER BY t.createdAt ASC")
List<Task> findNextPendingTasks(Pageable pageable);
// PROBLEM: If Worker A calls this, it locks the selected rows.
// When Worker B calls it, it will block and wait until Worker A's transaction completes.
// This serializes access and kills concurrency.
```

---

## Additional Best Practices

### Performance Considerations:
- **Index your query columns:** Ensure proper indexes exist on columns used in `WHERE`, `ORDER BY`, and `JOIN` clauses.
- **Use LIMIT appropriately:** When processing work queues, always use `LIMIT` to control batch size and prevent memory issues.
- **Monitor lock contention:** In high-concurrency scenarios, monitor your database for lock contention and adjust batch sizes accordingly.

### Testing Native Queries:
- **Test with actual data volumes:** Native queries often perform differently with large datasets.
- **Verify parameter binding:** Always test that your parameters are properly bound and SQL injection is prevented.
- **Test concurrency scenarios:** For `SKIP LOCKED` patterns, test with multiple concurrent workers to ensure proper behavior.