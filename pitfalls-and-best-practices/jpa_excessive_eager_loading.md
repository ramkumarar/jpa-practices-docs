# JPA Excessive Eager Loading

## The Problem

Using `fetch = FetchType.EAGER` on associations, especially collections, is a major performance anti-pattern. It forces JPA to load associated entities from the database whether they are needed or not. This leads to several critical issues:

1.  **Over-fetching**: It loads potentially large graphs of objects into memory, consuming significant resources and slowing down queries, even when the application only needed a single field from the root entity.
2.  **N+1 Select Problem in Disguise**: When fetching multiple root entities (e.g., via `findAll()`), `EAGER` on a collection association often results in N+1 queries—one query to load the root entities, and then N separate queries to load the collection for each root entity.
3.  **Cartesian Products**: If multiple collection associations are marked as `EAGER`, JPA generates a single SQL query with multiple `JOIN`s, resulting in a Cartesian Product. This can cause the database to return an enormous, duplicated result set, which is both slow and memory-intensive.
4.  **Hiding Lazy Loading Issues**: Eager fetching can mask underlying design problems and the need for proper DTO projections or query-specific fetching strategies. It often goes hand-in-hand with the **Open Session in View** anti-pattern, which keeps the database session open until the view is rendered, hiding `LazyInitializationException`s at the cost of poor performance and resource management.

## Why This Happens

`FetchType.EAGER` is an instruction to the JPA provider (like Hibernate) to "always load this association immediately."

-   For a to-one association (`@ManyToOne`, `@OneToOne`), this adds a `JOIN` to the original query.
-   For a to-many association (`@OneToMany`, `@ManyToMany`), Hibernate has two strategies:
    1.  **Subselect**: It can issue a second, separate `SELECT` to fetch the collections for all the root entities loaded in the first query.
    2.  **Join**: It can use a `LEFT OUTER JOIN` to fetch everything at once. This is what leads to the Cartesian Product problem if multiple collections are joined.

The default fetch type for `@OneToMany` and `@ManyToMany` is `LAZY`, but for `@ManyToOne` and `@OneToOne` it is `EAGER`. Developers often override this to `EAGER` on collections, thinking it will solve `LazyInitializationException`s, but in doing so, they create a much worse performance problem.

## Real-World Example

Consider an `Author` entity with two eagerly fetched collections.

```java
@Entity
public class Author {
    @Id
    private Long id;
    private String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.EAGER) // Problem 1
    private Set<Post> posts = new HashSet<>();

    @ManyToMany(fetch = FetchType.EAGER) // Problem 2
    @JoinTable(...)
    private Set<Award> awards = new HashSet<>();
}
```

When you execute `authorRepository.findById(1L)`, Hibernate generates a query like this:

```sql
-- Cartesian Product Query
SELECT ...
FROM author a
LEFT OUTER JOIN post p ON a.id = p.author_id
LEFT OUTER JOIN author_award aa ON a.id = aa.author_id
LEFT OUTER JOIN award aw ON aa.award_id = aw.id
WHERE a.id = ?
```

If an author has 10 posts and 5 awards, the database returns 10 * 5 = 50 rows, from which Hibernate must reconstruct the object graph. This is incredibly inefficient.

## Solutions

The solution is to default to lazy loading and fetch data on a per-use-case basis.

### 1. Default to `FetchType.LAZY` (Recommended)

This should be your default for **all** associations.

```java
@Entity
public class Author {
    @Id
    private Long id;

    // Good: LAZY is the default, but it's good to be explicit
    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private Set<Post> posts = new HashSet<>();

    // Good: Always make to-many associations LAZY
    @ManyToMany(fetch = FetchType.LAZY)
    private Set<Award> awards = new HashSet<>();

    // Good: Also make to-one associations LAZY
    @ManyToOne(fetch = FetchType.LAZY)
    private Publisher publisher;
}
```

### 2. Use `JOIN FETCH` or `@EntityGraph` for Specific Queries

When a specific business use case requires an association, fetch it explicitly.

**Using `JOIN FETCH` in JPQL:**
```java
@Repository
public interface AuthorRepository extends JpaRepository<Author, Long> {
    @Query("SELECT a FROM Author a JOIN FETCH a.posts WHERE a.id = :id")
    Optional<Author> findByIdWithPosts(@Param("id") Long id);
}
```

**Using `@EntityGraph`:**
`@EntityGraph` provides a cleaner way to specify which associations to fetch.

```java
@Repository
public interface AuthorRepository extends JpaRepository<Author, Long> {
    @EntityGraph(attributePaths = { "posts", "awards" })
    Optional<Author> findById(Long id);
}
```

### 3. Use DTO Projections for Read-Only Scenarios

This is the most performant solution for read operations. It fetches only the columns you need directly into a DTO, completely bypassing the entity and its associations.

```java
// DTO Class
public class AuthorDetailsDto {
    private String name;
    private int postCount;
    // constructor, getters
}

// Repository method
@Query("SELECT new com.example.AuthorDetailsDto(a.name, size(a.posts)) FROM Author a WHERE a.id = :id")
Optional<AuthorDetailsDto> findAuthorDetails(@Param("id") Long id);
```

Spring Data Projections (interfaces or records) offer an even simpler way to achieve this.

## Best Practices

1.  **LAZY is Your Best Friend**: Make `FetchType.LAZY` the default for all associations.
2.  **Disable Open Session in View**: Turn off `spring.jpa.open-in-view` in your configuration. This will force you to deal with `LazyInitializationException`s properly by using `JOIN FETCH`, `@EntityGraph`, or DTOs, rather than hiding the problem.
3.  **Fetch Per Query**: Analyze each use case. If you need related data, write a specific query for it. Don't rely on a global `EAGER` setting.
4.  **DTOs for Read, Entities for Write**: Use DTO projections for any data that is read-only. Load full entities only when you intend to modify them.
5.  **Beware of `toString()`**: An `EAGER` association can cause a `toString()` method to trigger a storm of lazy loading queries if the associated entities are not already fetched.

## Detection and Testing

1.  **SQL Logging**: Enable `spring.jpa.show-sql=true`. Look for queries with multiple `LEFT OUTER JOIN`s, especially when you only called a simple `findById` or `findAll`. A high number of `SELECT` statements after an initial one is a clear sign of an N+1 problem.
2.  **Performance Profiling**: Use a profiler like VisualVM or YourKit to identify slow queries and high memory allocation. Eager fetching is a common culprit.
3.  **Assert Query Counts**: Use a library like **datasource-proxy** or a custom `TestExecutionListener` to write tests that assert the exact number of queries being executed.
    ```java
    @Test
    @Transactional
    public void whenFindingAuthor_thenPostsShouldBeLazy() {
        // Given an author with posts
        // ...

        QueryCountHolder.start();
        Optional<Author> authorOpt = authorRepository.findById(authorId);
        QueryCount queryCount = QueryCountHolder.stop();

        // Then assert only one query was made for the author
        assertEquals(1, queryCount.getSelect());

        // And assert the collection is not initialized
        assertFalse(Hibernate.isInitialized(authorOpt.get().getPosts()));
    }
    ```

---

## Key Takeaway

> **Default to `FetchType.LAZY` for all associations.** Eager fetching leads to over-fetching, N+1 queries, and Cartesian Products. Fetch data explicitly on a per-query basis using `JOIN FETCH`, `@EntityGraph`, or DTO projections when you actually need it.

---

## See Also

-   [The N+1 Query Problem](./jpa_n_plus_1_problem.md)
-   [Improper Transaction Management (and Open Session in View)](./jpa_transaction_management.md)