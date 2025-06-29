# JPA Missing JDBC Batching Configuration

## The Problem

By default, most JPA providers, including Hibernate, do not enable JDBC batching. This means that even when using methods like `saveAll()`, the application sends each SQL statement (`INSERT`, `UPDATE`, `DELETE`) to the database in a separate network roundtrip.

This leads to significant performance degradation due to high network latency and increased database overhead. The application fails to leverage the database's ability to process multiple statements in a single, optimized batch.

## Why This Happens

JDBC batching is not automatic and fails to engage for several common reasons:

1.  **Default Configuration**: Hibernate's `hibernate.jdbc.batch_size` property is not set by default. Without this property, Hibernate's batching mechanism is completely disabled.
2.  **`IDENTITY` Primary Key Generation**: When an entity uses the `@GeneratedValue(strategy = GenerationType.IDENTITY)`, Hibernate is forced to execute the `INSERT` statement immediately to retrieve the database-generated primary key. This fundamentally prevents batching of `INSERT` statements.
3.  **JDBC Driver Requirements**: Some JDBC drivers have specific requirements to enable true batching. For example, the MySQL JDBC driver requires the `rewriteBatchedStatements=true` parameter in the JDBC connection URL to actually group statements into a single network packet. Without it, the driver sends statements one by one, even if Hibernate has prepared a batch.
4.  **Lack of Statement Ordering**: For optimal batching, Hibernate can group similar statements together (e.g., all `INSERT`s for `Post` entities). This is controlled by the `hibernate.order_inserts` and `hibernate.order_updates` properties, which are disabled by default.

## Real-World Example

Consider saving a large number of new entities to the database.

```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // This is a key part of the problem
    private Long id;
    private String name;
    // ... constructors, getters, setters
}

// The code that should be batching
@Transactional
public void createProducts(List<String> productNames) {
    List<Product> products = new ArrayList<>();
    for (String name : productNames) {
        products.add(new Product(name));
    }
    productRepository.saveAll(products); // Looks efficient, but isn't
}
```

Without proper configuration, the `saveAll` call will result in the following behavior, as seen in the SQL logs:

```sql
-- SQL Log (Batching Disabled)
Hibernate: insert into product (name) values (?)
Hibernate: insert into product (name) values (?)
Hibernate: insert into product (name) values (?)
-- ... 97 more individual inserts
```

Each `insert` statement is a separate database roundtrip, creating a massive performance bottleneck.

## Solutions

To enable JDBC batching, you must explicitly configure your JPA provider and potentially your JDBC driver.

### Recommended Solution: Configure Batching Properties

In a Spring Boot application, configure the necessary Hibernate properties in `application.yml` or `application.properties`.

**For `application.yml`:**
```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50  # Enable batching with a size of 50
        order_inserts: true # Order inserts for better batching
        order_updates: true # Order updates for better batching
        generate_statistics: true # Optional: for monitoring
  datasource:
    # For MySQL, this is critical for actual batch performance
    url: jdbc:mysql://localhost:3306/mydb?rewriteBatchedStatements=true
```

**For `application.properties`:**
```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.generate_statistics=true

# For MySQL
spring.datasource.url=jdbc:mysql://localhost:3306/mydb?rewriteBatchedStatements=true
```

### Change the ID Generation Strategy

To enable batching for `INSERT` statements, you must change the ID generation strategy from `IDENTITY` to `SEQUENCE`.

```java
@Entity
public class Product {
    @Id
    // Use SEQUENCE to allow batching
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(name = "product_seq", sequenceName = "product_sequence", allocationSize = 50)
    private Long id;
    private String name;
    // ...
}
```
The `allocationSize` in `@SequenceGenerator` should ideally match the `batch_size` for maximum efficiency.

## Best Practices

1.  **Always Set `batch_size`**: A `batch_size` between 30 and 50 is a reasonable starting point for most applications.
2.  **Use `SEQUENCE` over `IDENTITY`**: If you need to perform bulk inserts, `SEQUENCE` is the superior choice for primary key generation.
3.  **Enable Statement Ordering**: Always set `order_inserts` and `order_updates` to `true` to allow Hibernate to create more efficient batches.
4.  **Verify JDBC Driver Behavior**: Check the documentation for your specific JDBC driver to see if it requires special configuration parameters like `rewriteBatchedStatements=true`.
5.  **Monitor and Tune**: Use Hibernate statistics or a JDBC proxy driver to verify that batching is working as expected and tune the `batch_size` based on performance testing.

## Detection and Testing

### Verifying Batching is Active

1.  **Hibernate Statistics**: Enable `hibernate.generate_statistics` and monitor the logs. You should see metrics like `HHH000038: 200 statements executed as 4 batches (avg 50.0000)` which confirms batching is active.
2.  **SQL Logging**: With batching enabled, the SQL logs will look different. Hibernate often logs the batch execution rather than every single statement.
3.  **JDBC Proxy Drivers**: Use tools like **p6spy** or **datasource-proxy** to get detailed logs about how many statements are being sent to the database in each transaction.

### Performance Testing

The most definitive way to test is to measure the performance difference.

```java
@Test
public void testBulkInsertPerformance() {
    List<String> productNames = new ArrayList<>();
    for (int i = 0; i < 1000; i++) {
        productNames.add("Product " + i);
    }

    long startTime = System.currentTimeMillis();
    productService.createProducts(productNames);
    long endTime = System.currentTimeMillis();

    System.out.println("Execution time: " + (endTime - startTime) + "ms");
    // Run this test with and without batching configuration to see the dramatic difference.
}
```
You should observe a significant drop in execution time when batching is correctly configured.

---

## Key Takeaway

> **JDBC batching is not enabled by default.** To achieve significant performance gains, you must set `spring.jpa.properties.hibernate.jdbc.batch_size`, enable statement ordering, and use the `SEQUENCE` identity generator instead of `IDENTITY`.

---

## See Also

-   [Inefficient Bulk Operations](./jpa_inefficient_bulk_operations.md)
-   [Identifier Generation Problems](./jpa_identifier_generation_problem.md)