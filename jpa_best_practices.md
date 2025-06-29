# JPA and Hibernate: A Summary of Best Practices

This document provides a high-level summary of key best practices for building high-performance, maintainable applications with JPA and Hibernate. Each point includes a link to a more detailed document that explains the underlying pitfalls and solutions.

---

## 1. Entity Design and Mapping

Correctly designing and mapping your entities is the foundation of a healthy JPA application.

-   **Use TSID for High-Performance IDs**: For modern, high-performance applications, prefer application-level, time-sorted identifiers (TSIDs). They enable efficient batching, prevent database index fragmentation, and simplify `equals()`/`hashCode()` implementations.
    -   *Learn more: [High-Performance IDs with TSID](./pitfalls-and-best-practices/jpa_tsid_identifiers.md)*

-   **If not using TSID, prefer `SEQUENCE` over `IDENTITY`**: Avoid the `IDENTITY` strategy, as it disables JDBC batching. `SEQUENCE` is a better database-centric choice but is less performant than TSID.
    -   *Learn more: [Inefficient Identifier Generation](./pitfalls-and-best-practices/jpa_identifier_generation_problem.md)*

-   **Implement `equals()` and `hashCode()` Correctly**: If using database-generated keys, you must handle `null` IDs and proxies carefully. The safest approach is to use a natural, immutable business key. **Using TSIDs solves this problem entirely**, as the ID is available and final upon object creation.
    -   *Learn more: [JPA `equals()` and `hashCode()` Pitfalls](./pitfalls-and-best-practices/jpa_equals_hashcode.md)*

-   **Prefer Bidirectional `@OneToMany` Associations**: Unidirectional `@OneToMany` mappings are inefficient and lead to extra `UPDATE` statements or an unwanted join table. Always map the relationship from both sides, with the `@ManyToOne` side owning the foreign key.
    -   *Learn more: [Inefficient Association Mappings](./pitfalls-and-best-practices/jpa_inefficient_association_mappings.md)*

-   **Use `Set` for `@ManyToMany` Collections**: Using a `List` for a many-to-many relationship causes Hibernate to delete and re-insert the entire collection on every modification, which is a catastrophic performance issue.
    -   *Learn more: [Inefficient Association Mappings](./pitfalls-and-best-practices/jpa_inefficient_association_mappings.md)*

-   **Use `@Version` for Concurrency Control**: Prevent lost updates in multi-user scenarios by enabling optimistic locking with the `@Version` annotation on all updatable entities.
    -   *Learn more: [Missing Optimistic Locking](./pitfalls-and-best-practices/jpa_optimistic_locking_pitfall.md)*

-   **Use `@DynamicUpdate` for "Wide" Entities**: For entities with many columns, use `@DynamicUpdate` to generate leaner `UPDATE` statements that only include changed columns.
    -   *Learn more: [Missing `@DynamicUpdate` Annotation](./pitfalls-and-best-practices/jpa_missing_dynamicupdate_annotation.md)*

-   **Handle Enums and JSON Correctly**: Always map enums using `@Enumerated(EnumType.STRING)`. For semi-structured data, leverage your database's native JSON type (e.g., `jsonb` in PostgreSQL) for efficient querying and storage.
    -   *Learn more: [Handling Enums and JSON Data](./pitfalls-and-best-practices/jpa_advanced_data_types.md)*

-   **Standardize on UTC for Timestamps**: Always store timestamps in UTC. Use `java.time.Instant` in your entities, a `TIMESTAMP WITH TIME ZONE` column in your database, and configure Hibernate's timezone to UTC to prevent timezone-related bugs.
    -   *Learn more: [Handling Date and Time with Timezones](./pitfalls-and-best-practices/jpa_temporal_best_practices.md)*

---

## 2. Querying and Data Fetching

How you fetch data is the most common source of performance problems.

-   **Default to `FetchType.LAZY`**: Eager fetching is a major anti-pattern that leads to over-fetching and N+1 queries. Make all associations lazy by default.
    -   *Learn more: [Excessive Eager Loading](./pitfalls-and-best-practices/jpa_excessive_eager_loading.md)*

-   **Solve N+1 Problems with Fetch Joins**: When you do need to load associations, fetch them explicitly in a single query using `JOIN FETCH` or `@EntityGraph`.
    -   *Learn more: [The N+1 Query Problem](./pitfalls-and-best-practices/jpa_n_plus_1_problem.md)*

-   **Separate Pagination from Collection Fetching**: Never use `JOIN FETCH` on a collection in the same query that you are trying to paginate. This forces Hibernate to paginate in memory. Use the two-query approach instead (fetch IDs, then fetch content).
    -   *Learn more: [JOIN FETCH with Pagination Pitfall](./pitfalls-and-best-practices/jpa_join_fetch_pagination_pitfall.md)*

-   **Use Projections (DTOs) for Read-Only Data**: Never return entities from read-only queries. Use DTOs or interface-based projections to fetch only the data you need, which is faster and more memory-efficient.
    -   *Learn more: [Inefficient DTO Projections](./pitfalls-and-best-practices/jpa_inefficient_dto_projections.md)*

-   **Use `getReferenceById()` for Associations**: When you only need to set a foreign key relationship, use `getReferenceById()` (formerly `getOne()`) to get a lightweight proxy without executing a `SELECT` query.
    -   *Learn more: [Improper Use of `getOne()` vs `findById()`](./pitfalls-and-best-practices/jpa_improper_getone_vs_findbyid.md)*

-   **Use `Specification` for Dynamic Queries**: Avoid building queries with string concatenation. Use the `Specification` interface for type-safe, maintainable, and secure dynamic query building.
    -   *Learn more: [Dynamic Queries with Specification](./pitfalls-and-best-practices/jpa_dynamic_queries_with_specification.md)*

-   **Leverage Native Queries and Database Views**: For complex reports or queries that use advanced, database-specific features (like CTEs or window functions), drop down to native SQL. For reusable complex reads, map a read-only entity to a database view.
    -   *Learn more: [Native Query Best Practices](./pitfalls-and-best-practices/jpa_native_query_best_practices.md)*
    -   *Learn more: [Using Database Views for Read Operations](./pitfalls-and-best-practices/jpa_database_views_for_read_operations.md)*

---

## 3. Performance and Transactions

Efficiently managing transactions and batch operations is key to a scalable application.

-   **Keep Transactions Short**: A transaction should be as short as possible. Never include external service calls (e.g., REST APIs, message queues) or slow I/O operations within a transactional boundary.
    -   *Learn more: [Improper Transaction Management](./pitfalls-and-best-practices/jpa_transaction_management.md)*

-   **Disable Open Session in View**: This pattern holds database connections open for the entire HTTP request, leading to connection pool exhaustion. Always set `spring.jpa.open-in-view=false`.
    -   *Learn more: [Open Session in View Anti-Pattern](./pitfalls-and-best-practices/jpa_open_session_in_view_pitfall.md)*

-   **Use `@Modifying` for Bulk Operations**: To update or delete data in bulk, use `@Modifying` queries. This is orders of magnitude faster than loading entities into memory and iterating over them.
    -   *Learn more: [Inefficient Bulk Operations](./pitfalls-and-best-practices/jpa_inefficient_bulk_operations.md)*

-   **Enable JDBC Batching**: For high-performance writes, you must enable JDBC batching in your configuration. This requires using the `SEQUENCE` ID generator.
    -   *Learn more: [Missing JDBC Batching Configuration](./pitfalls-and-best-practices/jpa_missing_jdbc_batching_configuration.md)*

-   **Avoid `CascadeType.REMOVE`**: This can trigger massive and slow delete operations. Handle deletions explicitly in your service layer with bulk queries.
    -   *Learn more: [Dangers of `CascadeType.REMOVE`](./pitfalls-and-best-practices/jpa_cascading_remove_pitfall.md)*

-   **Manage the Persistence Context Size**: Avoid loading thousands of entities into the persistence context, as this causes high memory usage and slow dirty checking. Use pagination or stateless sessions for batch processing.
    -   *Learn more: [Persistence Context Bloat](./pitfalls-and-best-practices/jpa_persistence_context_bloat.md)*

---

## 4. Testing

-   **Test Against a Real Database**: Use **Testcontainers** to run your integration tests against a real, ephemeral database instance (like PostgreSQL) instead of H2.
-   **Isolate Tests**: Use `@Sql` scripts to clean and prepare the database state between tests to prevent flaky results.
-   **Understand Test Annotations**: Know that `@DataJpaTest` rolls back transactions by default and replaces your datasource, while `@SpringBootTest` does not. Use `@AutoConfigureTestDatabase(replace = Replace.NONE)` to force `@DataJpaTest` to use your real (Testcontainer) database.
    -   *Learn more: [JPA Testing Pitfalls and Best Practices](./pitfalls-and-best-practices/jpa_testing_pitfalls.md)*

-   **Use a Migration Tool for Schema Management**: Never use `ddl-auto: update` in production. Manage all schema changes with a dedicated tool like **Liquibase** to ensure safe, version-controlled, and repeatable migrations.
    -   *Learn more: [Database Schema Management](./pitfalls-and-best-practices/jpa_schema_management_best_practices.md)*
