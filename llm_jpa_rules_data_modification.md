---
description: "LLM rules for best practices in JPA data modification and transaction management."
alwaysApply: true
---
# JPA Data Modification and Transaction Rules

## Rule 1: Use Fetch-Then-Modify for Updates
**Title:** For Updates, Fetch the Entity First, then Modify It
**Description:** The most reliable way to update an entity is to first load it from the database within a transaction. This returns a managed instance. Modify this managed instance directly and let Hibernate's dirty checking handle the `UPDATE` statement automatically when the transaction commits. Do not call `save()` on a detached object.

**Good Example:**
```java
import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class ProductService {
    private final ProductRepository productRepository;

    @Transactional
    public void renameProduct(Long productId, String newName) {
        // 1. Fetch the entity, it becomes managed.
        Product product = productRepository.findById(productId).orElseThrow();

        // 2. Modify the managed entity.
        product.setName(newName);

        // 3. No 'save()' call is needed. The change is flushed automatically.
    }
}
```

**Bad Example:**
```java
import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class ProductService {
    private final ProductRepository productRepository;

    @Transactional
    public void renameProduct(Product detachedProduct) {
        // PROBLEM: Calling save() on a detached object performs a 'merge'.
        // The original 'detachedProduct' object remains unmanaged, and
        // this pattern is confusing and can hide bugs.  Furthermore, it bypasses Hibernate’s versioning mechanisms, potentially leading to lost updates if multiple users modify the same entity concurrently.
        productRepository.save(detachedProduct);
    }
}
```

---

## Rule 2: Keep Transactions Short and Focused
**Title:** Keep Transactions Short and Focused on Database Operations
**Description:** A transaction should only wrap the code that directly interacts with the database. Move any long-running, non-database logic (like external API calls or file I/O) outside of the transactional block. To achieve this, refactor the transactional logic into its own service.

**Good Example (Refactored into two services):**
```java
import lombok.RequiredArgsConstructor;

// Orchestration Service (non-transactional)
@Service
@RequiredArgsConstructor
public class OrderProcessingService {
    private final ExternalApiClient externalApiClient;
    private final OrderPersistenceService orderPersistenceService;

    public void processOrder(OrderData data) {
        // 1. Perform non-transactional logic first.
        ApiResponse validation = externalApiClient.validate(data);
        if (!validation.isSuccess()) {
            throw new ValidationException("Invalid order");
        }

        // 2. Call the dedicated transactional service. This call goes through
        //    the Spring proxy, correctly starting a new transaction.
        orderPersistenceService.saveOrder(data);
    }
}

import lombok.RequiredArgsConstructor;

// Persistence Service (transactional)
@Service
@RequiredArgsConstructor
public class OrderPersistenceService {
    private final OrderRepository orderRepository;
    private final InventoryRepository inventoryRepository;

    @Transactional
    public void saveOrder(OrderData data) {
        Order order = new Order(data);
        orderRepository.save(order);
        inventoryRepository.updateStock(data.getProductId());
    }
}
```
*Note: Using a `TransactionTemplate` programmatically within a single method is also an excellent alternative to achieve a fine-grained transaction scope.*

**Bad Example:**
```java
@Transactional // PROBLEM: This transaction is held open during the slow network call.
public void processOrder(OrderData data) {
    Order order = orderRepository.save(new Order(data));
    
    // This external API call could take seconds, holding the DB connection hostage.
    externalApiClient.validate(data);
    
    inventoryRepository.updateStock(data.getProductId());
}
```

---

## Rule 3: Use `readOnly = true` for All Read Operations
**Title:** Use `@Transactional(readOnly = true)` for All Read Operations
**Description:** For methods that only read data, always use `@Transactional(readOnly = true)`. This provides a significant performance optimization by signaling to the persistence provider that it can skip dirty checking and other overhead associated with write transactions. It's good practice to set this at the class level for query-focused services.

**Good Example:**
```java
import lombok.RequiredArgsConstructor;

@Service
@Transactional(readOnly = true) // Good default for a query-heavy service.
@RequiredArgsConstructor
public class ProductQueryService {

    private final ProductRepository productRepository;

    public ProductDTO findProduct(Long id) {
        // This method inherits readOnly = true.
        return productRepository.findProductDTOById(id);
    }

    @Transactional // Override the default for the rare write operation.
    public void updateProduct(...) {
        // ...
    }
}
```

**Bad Example:**
A service method that only performs `SELECT` queries but is annotated with a default, read-write `@Transactional`.

---

## Rule 4: Avoid Transactional Self-Invocation
**Title:** Avoid Calling a `@Transactional` Method from Within the Same Class
**Description:** Spring's transaction management works via proxies. Calling a transactional method from another method within the same class (e.g., `this.myTransactionalMethod()`) bypasses the proxy, and no transaction will be started for that specific method call. The best solution is to have a single public, transactional method that orchestrates private, non-transactional helper methods, or to refactor into separate services as shown in Rule 2.

**Good Example:**
```java
import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;

    @Transactional
    public void processUsers(List<UserData> userList) {
        for (UserData user : userList) {
            // Correct: Call a private helper method. It runs inside the
            // processUsers transaction.
            createUser(user);
        }
    }

    // This is just a helper method, not intended to be transactional on its own.
    private void createUser(UserData data) {
        User user = new User(data.getName());
        userRepository.save(user);
    }
}
```

**Bad Example:**
```java
import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;

    public void processUsers(List<UserData> userList) {
        for (UserData user : userList) {
            // PROBLEM: This call bypasses the Spring proxy.
            // The @Transactional on createUser is ignored.
            this.createUser(user);
        }
    }

    @Transactional
    public void createUser(UserData data) {
        User user = new User(data.getName());
        userRepository.save(user);
    }
}
```

---

## Rule 5: Configure Transaction Rollback Behavior
**Title:** Explicitly Configure Rollback Rules for Transactions
**Description:** By default, Spring only rolls back transactions for unchecked exceptions (`RuntimeException` and `Error`). It will commit a transaction if a checked exception is thrown. This is rarely the desired behavior. Always define your rollback policy explicitly.

**Good Example:**
```java
// This transaction will roll back for ANY exception, which is the safest default.
@Transactional(rollbackFor = Exception.class)
public void createUser(User user) throws UserCreationException {
    try {
        userRepository.save(user);
        // The INSERT statement is sent to the DB upon transaction commit/flush.
        // If the DB constraint fails then, it throws DataIntegrityViolationException.
        
    } catch (DataIntegrityViolationException e) {
        // The transaction is already marked for rollback by Spring.
        // This catch block is for translating the generic exception into a
        // more specific, domain-level exception for the caller.
        throw new UserAlreadyExistsException("User already exists", e);
    }
}
```

**Bad Example (Commits on Checked Exception):**
```java
@Transactional // No rollbackFor attribute.
public void createUser(User user) throws java.io.IOException {
    userRepository.save(user);
    // If this method throws a checked IOException, the transaction will
    // INCORRECTLY COMMIT the user save, leaving data in an inconsistent state.
    fileLogger.log(user.toString()); 
}
```

**Bad Example (Swallowing Exception):**
```java
@Transactional
public void createUser(User user) {
    try {
        userRepository.save(user);
    } catch (DataIntegrityViolationException e) {
        // PROBLEM: The exception is caught and logged, but not re-thrown.
        // The transaction will COMMIT successfully because the exception
        // never reached the Spring transaction proxy. This hides the failure.
        log.error("User already exists", e);
    }
}
```
---

## Rule 6: Use `getReferenceById()` for Setting Associations
**Title:** Use `getReferenceById()` Instead of `findById()` When Only Setting a Foreign Key
**Description:** When associating entities, you often only need a reference to an entity, not its actual data. Calling `findById()` executes an unnecessary `SELECT` query to fetch the full entity. Instead, use `getReferenceById()` which returns a lightweight proxy without hitting the database. This proxy is sufficient to establish the foreign key relationship when the parent entity is saved. Only use `findById()` when you need to access the entity's data.

**Good Example (Efficiently setting a relationship):**
```java
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;
    private final AuthorRepository authorRepository;

    @Transactional
    public void createPost(Long authorId, String content) {
        // 1. No database call here. A lightweight proxy is created.
        Author authorProxy = authorRepository.getReferenceById(authorId);

        Post post = new Post(content);
        post.setAuthor(authorProxy); // 2. Set the relationship

        // 3. A single INSERT statement is executed for the post.
        postRepository.save(post);
    }
}
```

**Bad Example (Inefficiently setting a relationship):**
```java
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;
    private final AuthorRepository authorRepository;

    @Transactional
    public void createPostWithInefficientAuthor(Long authorId, String content) {
        // 1. Unnecessary SELECT query is executed here
        Author author = authorRepository.findById(authorId)
            .orElseThrow(() -> new EntityNotFoundException("Author not found"));

        Post post = new Post(content);
        post.setAuthor(author); // 2. We only needed the author reference for this line
        postRepository.save(post);
    }
}
```
**Warning on `getReferenceById()`:** Be aware that accessing attributes of the proxy object (e.g., `authorProxy.getName()`) outside of an active transaction will throw a `LazyInitializationException`.

---