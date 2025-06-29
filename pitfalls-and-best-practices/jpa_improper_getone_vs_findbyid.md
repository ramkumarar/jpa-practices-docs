# JPA Improper Use of getOne() vs findById()

## The Problem

Developers often misuse `JpaRepository`'s methods for retrieving single entities, leading to performance issues or runtime errors. The two main anti-patterns are:

1.  **Using `findById()` when only a reference is needed**: This causes an unnecessary `SELECT` query to be executed against the database, when all that was required was to set a foreign key relationship on another entity.
2.  **Using `getOne()` / `getReferenceById()` incorrectly**: Developers may not understand that this method returns a lazy-loaded proxy. Accessing any attribute of this proxy (other than the ID) outside of an active transaction will result in a `LazyInitializationException`. Furthermore, if the underlying entity doesn't exist, an `EntityNotFoundException` is thrown only upon first access, which can be far from the original call site and harder to debug.

It's also critical to note that **`getOne()` is deprecated** in recent versions of Spring Data JPA and has been replaced by `getReferenceById()`.

## Why This Happens

The confusion stems from the fundamental difference in how these methods work:

-   **`findById(ID id)`**:
    -   **Behavior**: Eagerly fetches the entity from the database. It executes a `SELECT` statement immediately.
    -   **Return Value**: Returns an `Optional<T>`. If the entity is found, it's contained in the `Optional`; otherwise, `Optional.empty()` is returned.
    -   **Use Case**: When you need to read, modify, or check the state of the entity itself.

-   **`getOne(ID id)` / `getReferenceById(ID id)`**:
    -   **Behavior**: Lazily fetches a reference to the entity. It does **not** hit the database. Instead, it returns a Hibernate proxy object which is a placeholder containing only the provided ID.
    -   **Database Hit**: A `SELECT` query is only triggered when you access any method on the proxy object other than `getId()`.
    -   **Return Value**: Returns a proxy of type `T`.
    -   **Exception Handling**: If the entity with the given ID does not exist, this method does not fail. It throws an `EntityNotFoundException` only when the proxy is first accessed (e.g., by calling `proxy.getName()`).
    -   **Use Case**: When you only need to establish a relationship (set a foreign key) on another entity without loading the entire object.

## Real-World Example

Let's say we have `Post` and `Author` entities.

```java
// Inefficient use of findById()
@Transactional
public void createPostWithInefficientAuthor(Long authorId, String content) {
    // 1. Unnecessary SELECT query is executed here
    Author author = authorRepository.findById(authorId)
        .orElseThrow(() -> new EntityNotFoundException("Author not found"));

    Post post = new Post(content);
    post.setAuthor(author); // 2. We only needed the author reference for this line
    postRepository.save(post);
}

// Incorrect use of getReferenceById()
public void printAuthorName(Long authorId) {
    // 1. No database hit here, a proxy is created
    Author authorProxy = authorRepository.getReferenceById(authorId);

    // 2. Throws LazyInitializationException because the session is closed
    System.out.println(authorProxy.getName());
}
```

## Solutions

The solution is to choose the method that matches your specific need.

### Recommended Solution: Use the Right Method for the Job

**Use `getReferenceById()` when setting a relationship:**

```java
// Good: Use getReferenceById() for setting relationships
@Transactional
public void createPost(Long authorId, String content) {
    // 1. No DB call here. A lightweight proxy is created.
    Author authorProxy = authorRepository.getReferenceById(authorId);

    Post post = new Post(content);
    post.setAuthor(authorProxy); // 2. Set the relationship

    // 3. A single INSERT statement is executed for the post.
    // The author_id foreign key is set correctly.
    postRepository.save(post);
}
```

**Use `findById()` when you need the entity's data:**

```java
// Good: Use findById() when you need the entity data
@Transactional
public void updateAuthorName(Long authorId, String newName) {
    // 1. SELECT query is executed to fetch the author's data.
    Author author = authorRepository.findById(authorId)
        .orElseThrow(() -> new EntityNotFoundException("Author not found"));

    // 2. We need the actual entity to modify its state.
    author.setName(newName);
    authorRepository.save(author); // An UPDATE statement is executed on transaction commit.
}
```

## Best Practices

1.  **Default to `getReferenceById()` for Associations**: When associating entities, `getReferenceById()` is almost always the correct and most performant choice.
2.  **Use `findById()` for Data Access**: When you need to read, check, or modify the attributes of an entity, use `findById()` and properly handle the returned `Optional`.
3.  **Stay within a Transaction**: Remember that accessing a proxy from `getReferenceById()` requires an active Hibernate Session. Ensure such access happens within a `@Transactional` boundary to avoid `LazyInitializationException`.
4.  **Stop Using `getOne()`**: It is deprecated. Update your code to use `getReferenceById()`. The behavior is nearly identical, but `getReferenceById` has clearer semantics.
5.  **Understand the Exception Model**: Be prepared for `findById()` to return an empty `Optional`. Be prepared for `getReferenceById()` to throw an `EntityNotFoundException` later in the process if the ID is invalid.

## Detection and Testing

### Detection

1.  **Enable SQL Logging**: Set `spring.jpa.show-sql=true`. Look for `SELECT` statements that are immediately followed by a `save()` or `persist()` on a *different* entity. This is a strong indicator that `findById()` was used where `getReferenceById()` would have been sufficient.
    ```sql
    -- Log for inefficient relationship setting
    Hibernate: select author0_.id as id1_0_0_, author0_.name as name2_0_0_ from author author0_ where author0_.id=?
    Hibernate: insert into post (author_id, content) values (?, ?)
    ```
2.  **Code Reviews**: During reviews, actively question the use of `findById()` when the returned entity is only used in a `set...()` call to establish a relationship.

### Testing

**Test for correct proxy usage:**
```java
@Test
@Transactional // Ensure transaction is active for proxy access
public void whenSettingAuthor_shouldUseProxyAndNotSelect() {
    // Given an existing author
    Author existingAuthor = authorRepository.save(new Author("John Doe"));

    // Use a query counter to verify behavior
    QueryCountHolder.start();

    // When creating a post using a reference to the author
    postService.createPost(existingAuthor.getId(), "My new post");

    QueryCount queryCount = QueryCountHolder.stop();

    // Then assert that no SELECT statement was issued for the Author
    assertEquals(0, queryCount.getSelect());
    assertEquals(1, queryCount.getInsert()); // For the new Post
}
```

**Test for `LazyInitializationException`:**
```java
@Test(expected = LazyInitializationException.class)
public void whenAccessingProxyOutsideTransaction_shouldThrow() {
    // Given an existing author
    Author existingAuthor = authorRepository.save(new Author("John Doe"));

    // When getting a reference (this works)
    Author authorProxy = authorRepository.getReferenceById(existingAuthor.getId());

    // Then, accessing it outside a transaction should fail
    // (This test method is not @Transactional)
    authorProxy.getName();
}
```

---

## Key Takeaway

> Use `getReferenceById()` to efficiently set foreign key relationships without a database query. Use `findById()` only when you need to access the entity's actual data. Always access proxies from within a transaction.