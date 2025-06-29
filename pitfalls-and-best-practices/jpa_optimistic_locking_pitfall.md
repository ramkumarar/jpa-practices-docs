# JPA Pitfall: Missing Optimistic Locking

## The Problem

In a multi-user application, it's common for two users to attempt to update the same piece of data at the same time. Without a concurrency control mechanism, this can lead to a "lost update" scenario.

The sequence of events is as follows:
1.  **User A** reads a record (e.g., a product with price $100).
2.  **User B** reads the *same* record (price is still $100).
3.  **User A** updates the price to `$120` and saves. The database now correctly stores the price as $120.
4.  **User B**, who is still looking at the old data, updates the price to $110 and saves. This action overwrites User A's update.

The final price in the database is $110. **User A's update has been silently lost.** This is a critical data integrity issue that can lead to incorrect financial data, inconsistent state, and frustrated users.

## Why This Happens

This happens because standard web applications operate on a disconnected, "read-commit-write" model. The data is read in one transaction, displayed to the user, and then written back in a completely separate, new transaction.

Between the read and the write, the application has no "lock" on the data in the database. JPA, by default, does not prevent this. When the second transaction commits, the persistence provider simply overwrites the data, unaware that it was based on a stale state.

## Real-World Example

Consider a product inventory management system where two employees try to update a product's stock level.

**The Entity (without locking):**
```java
@Entity
public class Product {
    @Id private Long id;
    private String name;
    private int stockLevel;
    // ...
}
```

**The Service:**
```java
@Service
public class ProductService {
    @Autowired private ProductRepository productRepository;

    @Transactional
    public void updateStockLevel(Long productId, int newStockLevel) {
        Product product = productRepository.findById(productId).orElseThrow();
        // Simulate some processing time before the update
        try { Thread.sleep(100); } catch (InterruptedException e) {}
        product.setStockLevel(newStockLevel);
        // The save will overwrite whatever is in the database
    }
}
```
If two concurrent calls are made, `updateStockLevel(1L, 50)` and `updateStockLevel(1L, 40)`, the last one to complete will win, and the other update will be lost.

## Solution: Use `@Version` for Optimistic Locking

The standard JPA solution for this problem is **optimistic locking**. This mechanism does not block other users but instead assumes that conflicts are rare. It works by checking if the data has changed before committing an update.

To enable it, you simply add a field to your entity and annotate it with `@Version`.

```java
@Entity
public class Product {
    @Id private Long id;
    private String name;
    private int stockLevel;

    // ✅ Enable optimistic locking
    @Version
    private Long version;
}
```

**How it Works:**
1.  When you read a `Product` entity, Hibernate also retrieves its `version` number (e.g., `version = 1`).
2.  When you commit a transaction to update the `Product`, Hibernate automatically increments the version number (`version` becomes `2`).
3.  Crucially, it adds the original version number to the `WHERE` clause of the `UPDATE` statement.
    ```sql
    UPDATE product SET stock_level=?, version=? WHERE id=? AND version=?
    -- Parameters: [50, 2, 1L, 1]
    ```
4.  If another transaction had updated the product in the meantime, its version in the database would already be `2`. The `WHERE id=1 AND version=1` clause of our update will not find any rows to update.
5.  Hibernate sees that 0 rows were affected, understands that a conflict occurred, and throws an **`OptimisticLockingFailureException`** (or a more specific subclass like `ObjectOptimisticLockingFailureException`).

**Handling the Exception:**
Your service layer must be prepared to catch this exception and handle it gracefully, usually by informing the user that the data has changed and they should retry their action.

```java
@Service
public class ProductService {
    @Autowired private ProductRepository productRepository;

    @Transactional
    public void updateStockLevel(Long productId, int newStockLevel) {
        try {
            Product product = productRepository.findById(productId).orElseThrow();
            product.setStockLevel(newStockLevel);
            productRepository.save(product); // save() will trigger the version check on flush
        } catch (ObjectOptimisticLockingFailureException e) {
            // ✅ Handle the conflict
            throw new ConcurrentUpdateException("Product was updated by another user. Please try again.", e);
        }
    }
}
```

## Best Practices

1.  **Use `@Version` on All Updatable Entities**: Any entity that can be modified by concurrent users should have a `@Version` field. The field can be of type `Long`, `Integer`, or `java.sql.Timestamp`.
2.  **Handle the Exception Gracefully**: Never let an `OptimisticLockingFailureException` bubble up to the end-user as a generic error. Catch it and provide a clear, actionable message (e.g., "The data has changed since you loaded it. Please refresh and try again.").
3.  **Prefer Optimistic Locking**: For most web applications, optimistic locking is far more scalable and performant than pessimistic locking (which uses database-level `SELECT ... FOR UPDATE` locks). Use pessimistic locking only when you know conflicts will be very frequent and you must prevent them upfront.

---

## Key Takeaway

> **Always use the `@Version` annotation on entities that can be updated concurrently.** This enables optimistic locking, which protects against lost updates by throwing an exception if a conflict is detected. Your application must be designed to catch this exception and handle it gracefully.

---

## See Also

-   [Improper Transaction Management](./jpa_transaction_management.md)
