# JPA Best Practice: High-Performance IDs with TSID

## The Problem

Choosing the right primary key generation strategy is critical for performance, but traditional methods have significant drawbacks:

1.  **`GenerationType.IDENTITY`**: The most common strategy, but it completely **disables JDBC batch inserts**, leading to poor performance in bulk operations.
2.  **`GenerationType.SEQUENCE`**: A much better choice that allows batching, but it requires an extra database roundtrip to fetch new IDs from the sequence, adding latency.
3.  **`UUID`**: Generated in the application and works well with batching, but standard UUIDs are random. This leads to **severe database index fragmentation**, as new records are inserted at random places in the index B-tree, causing performance to degrade over time as the table grows.
4.  **`equals()` and `hashCode()`**: Database-generated keys (like `IDENTITY` and `SEQUENCE`) are `null` before an entity is persisted, which complicates the implementation of `equals()` and `hashCode()` and can lead to issues with `Set` collections.

## Solution: Use Time-Sorted Unique Identifiers (TSID)

A modern and highly effective solution is to use **Time-Sorted Unique Identifiers (TSIDs)**. A TSID is a 64-bit number that is time-sortable, object-sortable, and completely unique. It combines the benefits of database sequences and UUIDs while avoiding their major pitfalls.

This strategy, often highlighted by experts like Vlad Mihalcea, provides a robust foundation for high-performance JPA applications.

### Why TSID is a Superior Choice

1.  **High-Performance Batching**: TSIDs are generated entirely within the application *before* being sent to the database. This allows Hibernate to use JDBC batching for `INSERT` statements at full efficiency.
2.  **Excellent Database Index Locality**: This is the key advantage over UUIDs. Because TSIDs are generated sequentially based on time, new IDs are always greater than previous ones. This means new rows are appended to the end of the database index, which is the most efficient way to write, preventing fragmentation and ensuring high performance even for very large tables.
3.  **No Database Roundtrips**: Unlike `SEQUENCE`, there is no need to ask the database for an ID, eliminating network latency from the ID generation process.
4.  **Pre-Persistence Identity**: An entity has its final, unique ID the moment it is instantiated in Java. This greatly simplifies `equals()` and `hashCode()` implementations, as the ID is never null.
5.  **Scalability**: TSIDs are generated without coordination, making them perfect for distributed and horizontally-scaled systems.

## Implementation Example

Using TSIDs in a Spring Boot / JPA application is straightforward.

### 1. Add the Dependency

First, add the `tsid-creator` library to your project.

**Maven (`pom.xml`):**
```xml
<dependency>
    <groupId>com.github.f4b6a3</groupId>
    <artifactId>tsid-creator</artifactId>
    <version>5.2.4</version> <!-- Check for the latest version -->
</dependency>
```

### 2. Create a Custom Hibernate ID Generator

Create a custom generator class that implements Hibernate's `IdentifierGenerator` interface.

```java
import com.github.f4b6a3.tsid.TsidCreator;
import org.hibernate.engine.spi.SharedSessionContractImplementor;
import org.hibernate.id.IdentifierGenerator;

import java.io.Serializable;

public class TsidGenerator implements IdentifierGenerator {

    @Override
    public Serializable generate(SharedSessionContractImplementor session, Object object) {
        // Generate a new TSID and return it as a Long
        return TsidCreator.getTsid().toLong();
    }
}
```

### 3. Annotate the Entity

Now, use this custom generator on your entity's primary key field. The `@GenericGenerator` annotation is used to register and apply our custom class.

```java
import org.hibernate.annotations.GenericGenerator;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;

@Entity
public class Post {

    @Id
    @GeneratedValue(generator = "tsid")
    @GenericGenerator(name = "tsid", strategy = "com.example.TsidGenerator")
    private Long id;

    private String title;

    // A TSID-based ID is perfect for equals/hashCode
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Post post = (Post) o;
        // The ID is never null, even for a new, un-persisted entity.
        return id.equals(post.id);
    }

    @Override
    public int hashCode() {
        return id.hashCode();
    }
}
```

With this setup, every time you save a new `Post` entity, Hibernate will call your `TsidGenerator` to get a new ID before sending the `INSERT` statement to the database.

## Best Practices

1.  **Use for High-Volume Tables**: TSIDs are especially beneficial for tables that experience a high rate of inserts, as this is where index performance is most critical.
2.  **Store as `BIGINT` or `NUMBER`**: The 64-bit TSID fits perfectly into a standard `BIGINT` (PostgreSQL, MySQL) or `NUMBER(19)` (Oracle) column.
3.  **Simplify `equals()` and `hashCode()`**: With TSIDs, you can safely base your equality methods on the ID field, as it's guaranteed to be non-null and unique from the moment of creation.
4.  **Educate Your Team**: Since this is a less common strategy than `IDENTITY` or `SEQUENCE`, ensure your team understands why it's being used and the performance benefits it provides.

---

## Key Takeaway

> For high-performance applications, especially those with high-volume inserts, **use TSIDs as your primary key strategy.** They enable efficient JDBC batching, prevent database index fragmentation, and simplify entity identity management (`equals`/`hashCode`), making them a superior alternative to both `IDENTITY` and `UUID`.

---

## See Also

-   [Inefficient Identifier Generation Strategies](./jpa_identifier_generation_problem.md)
-   [Missing JDBC Batching Configuration](./jpa_missing_jdbc_batching_configuration.md)
-   [JPA `equals()` and `hashCode()` Implementation Issues](./jpa_equals_hashcode.md)
