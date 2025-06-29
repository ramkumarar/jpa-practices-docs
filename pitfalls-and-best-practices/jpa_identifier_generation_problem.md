# JPA Inefficient Identifier Generation Strategies

## The Problem

Inefficient identifier generation strategies can severely impact application performance by disabling JDBC batching, causing unnecessary database round trips, and creating transaction overhead. The most common problematic strategies are:

1. **IDENTITY Generator**: Disables JDBC batching because the ID is only available after INSERT execution
2. **TABLE Generator**: Uses separate transactions for ID generation, causing significant overhead
3. **AUTO Strategy**: Often defaults to inefficient strategies depending on the database

These issues become particularly pronounced during bulk operations where hundreds or thousands of entities need to be persisted.

## Why This Happens

### IDENTITY Strategy Issues
- **No Batching**: JDBC batching requires knowing all SQL statements upfront, but IDENTITY IDs are only available after execution
- **Immediate Execution**: Each INSERT must be executed immediately to get the generated ID
- **Database Round Trips**: One round trip per entity instead of batched execution

### TABLE Strategy Issues
- **Separate Transactions**: ID generation requires its own transaction to ensure uniqueness
- **Lock Contention**: Multiple threads competing for the same ID table row
- **Additional Queries**: Extra SELECT/UPDATE operations for each ID generation
- **Scalability Problems**: Single point of contention across the entire application

### AUTO Strategy Issues
- **Database-Dependent**: Behavior varies by database vendor
- **Often Defaults to IDENTITY**: Many databases default to less efficient strategies

## Real-World Example

### Problematic IDENTITY Setup
```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // Problem!
    private Long id;
    
    private String orderNumber;
    private BigDecimal amount;
    private LocalDateTime createdAt;
    
    // getters and setters
}

@Entity
@Table(name = "order_items")
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // Problem!
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;
    
    private String productName;
    private Integer quantity;
    private BigDecimal price;
    
    // getters and setters
}
```

### Problematic TABLE Setup
```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "product_gen")
    @TableGenerator(
        name = "product_gen",
        table = "id_generator",
        pkColumnName = "gen_name",
        valueColumnName = "gen_value",
        pkColumnValue = "product_id",
        allocationSize = 1 // Very inefficient!
    )
    private Long id;
    
    private String name;
    private BigDecimal price;
    
    // getters and setters
}
```

### Performance Impact Example
```java
@Service
@Transactional
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private OrderItemRepository orderItemRepository;
    
    // This will be VERY slow with IDENTITY strategy
    public void createBulkOrders(List<OrderDto> orderDtos) {
        List<Order> orders = new ArrayList<>();
        
        for (OrderDto dto : orderDtos) { // 1000 orders
            Order order = new Order();
            order.setOrderNumber(dto.getOrderNumber());
            order.setAmount(dto.getAmount());
            order.setCreatedAt(LocalDateTime.now());
            
            orders.add(order);
        }
        
        // With IDENTITY: 1000 individual INSERT statements
        // Without batching due to ID generation strategy
        orderRepository.saveAll(orders);
        
        // Even worse performance when adding items
        for (Order order : orders) {
            for (int i = 0; i < 5; i++) { // 5 items per order
                OrderItem item = new OrderItem();
                item.setOrder(order);
                item.setProductName("Product " + i);
                item.setQuantity(1);
                item.setPrice(BigDecimal.valueOf(10.00));
                
                orderItemRepository.save(item); // Another INSERT per item
            }
        }
        // Total: 6000 individual INSERT statements instead of 2 batched statements!
    }
}
```

### Generated SQL with IDENTITY (inefficient)
```sql
-- For each order (1000 times):
INSERT INTO orders (order_number, amount, created_at) VALUES (?, ?, ?);

-- For each order item (5000 times):
INSERT INTO order_items (order_id, product_name, quantity, price) VALUES (?, ?, ?, ?);

-- Total: 6000 individual INSERT statements
-- No batching possible due to IDENTITY strategy
```

## Solutions

### Use SEQUENCE Strategy with Optimizers

The best solution is to use the SEQUENCE generation strategy with proper allocation size optimization:

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
    @SequenceGenerator(
        name = "order_seq",
        sequenceName = "order_sequence",
        allocationSize = 50 // Pre-allocate 50 IDs at once
    )
    private Long id;
    
    private String orderNumber;
    private BigDecimal amount;
    private LocalDateTime createdAt;
    
    // getters and setters
}

@Entity
@Table(name = "order_items")  
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_item_seq")
    @SequenceGenerator(
        name = "order_item_seq",
        sequenceName = "order_item_sequence", 
        allocationSize = 100 // Higher allocation for frequent inserts
    )
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;
    
    private String productName;
    private Integer quantity;
    private BigDecimal price;
    
    // getters and setters
}
```

### Enable JDBC Batching (Required with SEQUENCE)
```properties
# application.properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.jdbc.batch_versioned_data=true
```

### Optimized SQL Generation with SEQUENCE
```sql
-- PostgreSQL - Create sequences with proper cache settings
CREATE SEQUENCE order_sequence START WITH 1000 INCREMENT BY 1 CACHE 50;
CREATE SEQUENCE order_item_sequence START WITH 10000 INCREMENT BY 1 CACHE 100;

-- Oracle - Similar syntax with cache
CREATE SEQUENCE order_sequence START WITH 1000 INCREMENT BY 1 CACHE 50;
CREATE SEQUENCE order_item_sequence START WITH 10000 INCREMENT BY 1 CACHE 100;

-- SQL Server
CREATE SEQUENCE order_sequence START WITH 1000 INCREMENT BY 1 CACHE 50;
CREATE SEQUENCE order_item_sequence START WITH 10000 INCREMENT BY 1 CACHE 100;
```

### Performance Comparison: Before vs After
```java
// Before (IDENTITY) - SQL execution pattern:
// INSERT INTO orders (order_number, amount, created_at) VALUES (?, ?, ?); -- 1000 times
// INSERT INTO order_items (order_id, product_name, quantity, price) VALUES (?, ?, ?, ?); -- 5000 times
// Total: 6000 individual statements, no batching possible

// After (SEQUENCE) - SQL execution pattern:
// SELECT NEXTVAL('order_sequence'); -- Once every 50 orders (20 times total)
// INSERT INTO orders (id, order_number, amount, created_at) VALUES (?, ?, ?, ?), (?, ?, ?, ?), ...; -- Batched
// SELECT NEXTVAL('order_item_sequence'); -- Once every 100 items (50 times total)  
// INSERT INTO order_items (id, order_id, product_name, quantity, price) VALUES (?, ?, ?, ?, ?), ...; -- Batched
// Total: ~72 statements with efficient batching
```

## Best Practices

### 1. SEQUENCE Strategy Configuration
```java
// For high-performance applications
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "high_perf_seq")
@SequenceGenerator(
    name = "high_perf_seq",
    sequenceName = "entity_sequence",
    allocationSize = 100 // Adjust based on insert frequency
)

// For moderate insert frequency
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "standard_seq")
@SequenceGenerator(
    name = "standard_seq",
    sequenceName = "entity_sequence",
    allocationSize = 50
)

// For low insert frequency  
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "low_freq_seq")
@SequenceGenerator(
    name = "low_freq_seq",
    sequenceName = "entity_sequence",
    allocationSize = 10
)
```

### 2. Database-Specific Sequence Creation
```sql
-- PostgreSQL
CREATE SEQUENCE order_sequence START WITH 1000 INCREMENT BY 1;
CREATE SEQUENCE order_item_sequence START WITH 10000 INCREMENT BY 1;

-- Oracle  
CREATE SEQUENCE order_sequence START WITH 1000 INCREMENT BY 1 CACHE 100;
CREATE SEQUENCE order_item_sequence START WITH 10000 INCREMENT BY 1 CACHE 200;

-- SQL Server
CREATE SEQUENCE order_sequence START WITH 1000 INCREMENT BY 1 CACHE 100;
```

### 3. Monitor Sequence Gap Issues
```java
@Component
public class SequenceMonitor {
    
    @Autowired
    private EntityManager entityManager;
    
    @Scheduled(fixedRate = 60000) // Every minute
    public void checkSequenceGaps() {
        String sql = "SELECT COUNT(*) FROM orders WHERE id BETWEEN :start AND :end";
        
        // Check for large gaps that might indicate allocation size too high
        long actualCount = entityManager.createQuery(sql, Long.class)
            .setParameter("start", 1000L)
            .setParameter("end", 2000L)
            .getSingleResult();
            
        if (actualCount < 500) { // Less than 50% utilization
            log.warn("Sequence allocation might be too high, causing gaps");
        }
    }
}
```

### 4. Batch Configuration Tuning
```java
@Configuration
public class JpaConfig {
    
    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        
        Properties properties = new Properties();
        properties.setProperty("hibernate.jdbc.batch_size", "50");
        properties.setProperty("hibernate.order_inserts", "true");
        properties.setProperty("hibernate.order_updates", "true");
        properties.setProperty("hibernate.jdbc.batch_versioned_data", "true");
        
        // For better batch processing
        properties.setProperty("hibernate.enable_lazy_load_no_trans", "false");
        properties.setProperty("hibernate.jdbc.lob.non_contextual_creation", "true");
        
        em.setJpaProperties(properties);
        return em;
    }
}
```

## Detection and Testing

### 1. SQL Query Logging and Analysis
```properties
# Enable detailed SQL logging
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.engine.jdbc.batch.internal.BatchingBatch=DEBUG
```

### 2. Performance Test for Bulk Operations
```java
@SpringBootTest
@Transactional
public class IdentifierGenerationPerformanceTest {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    public void testBulkInsertPerformance() {
        int entityCount = 1000;
        
        long startTime = System.currentTimeMillis();
        
        List<Order> orders = new ArrayList<>();
        for (int i = 0; i < entityCount; i++) {
            Order order = new Order();
            order.setOrderNumber("ORDER-" + i);
            order.setAmount(BigDecimal.valueOf(100.0));
            order.setCreatedAt(LocalDateTime.now());
            orders.add(order);
        }
        
        orderRepository.saveAll(orders);
        
        long endTime = System.currentTimeMillis();
        long duration = endTime - startTime;
        
        // Should complete in reasonable time
        assertTrue("Bulk insert should be fast with proper ID strategy", 
                   duration < 5000); // Less than 5 seconds for 1000 entities
        
        System.out.println("Bulk insert of " + entityCount + 
                          " entities took: " + duration + "ms");
    }
}
```

### 3. JDBC Batch Verification
```java
@Test
public void verifyBatchingIsEnabled() {
    SessionFactory sessionFactory = entityManager.getEntityManagerFactory()
        .unwrap(SessionFactory.class);
    
    Statistics stats = sessionFactory.getStatistics();
    stats.setStatisticsEnabled(true);
    stats.clear();
    
    // Perform bulk operation
    List<Order> orders = createTestOrders(100);
    orderRepository.saveAll(orders);
    
    // Verify batching occurred
    long batchExecutions = stats.getSuccessfulTransactionCount();
    assertTrue("JDBC batching should reduce number of executions", 
               batchExecutions < 10); // Much less than 100 individual inserts
}
```

### 4. Sequence Allocation Testing
```java
@Test
public void testSequenceAllocation() {
    // Clear any existing entities
    orderRepository.deleteAll();
    
    // Create first entity - should allocate sequence range
    Order order1 = new Order();
    order1.setOrderNumber("TEST-1");
    order1 = orderRepository.save(order1);
    
    Long firstId = order1.getId();
    
    // Create second entity - should use pre-allocated ID
    Order order2 = new Order();
    order2.setOrderNumber("TEST-2");
    order2 = orderRepository.save(order2);
    
    Long secondId = order2.getId();
    
    // IDs should be consecutive (within allocation size)
    assertTrue("Sequential IDs should be allocated from pool", 
               secondId.equals(firstId + 1));
}
```

### 5. Database Sequence Monitoring
```java
@Component
public class SequenceHealthCheck {
    
    @Autowired
    private EntityManager entityManager;
    
    public SequenceStatus checkSequenceHealth(String sequenceName) {
        String sql = "SELECT last_value, is_called FROM " + sequenceName;
        
        Object[] result = (Object[]) entityManager.createNativeQuery(sql)
            .getSingleResult();
            
        Long lastValue = (Long) result[0];
        Boolean isCalled = (Boolean) result[1];
        
        return new SequenceStatus(sequenceName, lastValue, isCalled);
    }
    
    public static class SequenceStatus {
        private final String name;
        private final Long lastValue;
        private final Boolean isCalled;
        
        // constructor and getters
    }
}
```

### 6. Load Testing with Different Strategies
```java
@Test
public void compareIdGenerationStrategies() {
    // Test with different strategies and measure performance
    Map<String, Long> results = new HashMap<>();
    
    // Test SEQUENCE strategy
    long sequenceTime = measureBulkInsert("SEQUENCE", 1000);
    results.put("SEQUENCE", sequenceTime);
    
    // Test IDENTITY strategy (if available)
    long identityTime = measureBulkInsert("IDENTITY", 1000);
    results.put("IDENTITY", identityTime);
    
    // SEQUENCE should be significantly faster
    assertTrue("SEQUENCE should outperform IDENTITY", 
               sequenceTime < identityTime * 0.5); // At least 2x faster
    
    results.forEach((strategy, time) -> 
        System.out.println(strategy + " strategy: " + time + "ms"));
}
```

---

## Key Takeaway

> **Use the `SEQUENCE` generation strategy with a properly configured `allocationSize` to enable JDBC batching.** Avoid `IDENTITY` for entities that require high-performance bulk inserts, as it disables batching entirely. The `TABLE` generator should also be avoided due to its poor performance and high contention.

---

## See Also

-   [Missing JDBC Batching Configuration](./jpa_missing_jdbc_batching_configuration.md)
-   [Inefficient Bulk Operations](./jpa_inefficient_bulk_operations.md)
