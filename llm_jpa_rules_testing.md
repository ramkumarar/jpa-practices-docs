---
description: "LLM rules for writing robust and reliable JPA tests."
alwaysApply: true
---
# JPA Testing Rules

## Rule 1: Use Testcontainers for Realistic Database Testing
**Title:** Always Test Against a Production-Like Database with Testcontainers
**Description:** Do not use in-memory databases like H2 for tests, as they behave differently from production databases (e.g., PostgreSQL, MySQL). Use Testcontainers to spin up a real database in a Docker container for your tests. You must also add `@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)` to prevent Spring from overriding your configuration with an in-memory one.

**Good Example (`@DataJpaTest`):**
```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class MyRepositoryTest {

    @Container
    private static final PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private MyRepository myRepository;

    @Test
    void testSomething() {
        // This test runs against a real PostgreSQL database.
    }
}
```

**Bad Example:**
```java
@DataJpaTest
// PROBLEM: This will use a default in-memory H2 database, which can hide bugs
// that would only appear on a real database like PostgreSQL.
class MyRepositoryTest {
    // ...
}
```

---

## Rule 2: Clean Up State in `@SpringBootTest`
**Title:** Use `@Sql` Scripts to Clean the Database Between `@SpringBootTest` Methods
**Description:** Unlike `@DataJpaTest`, `@SpringBootTest` does not automatically roll back transactions. This means data from one test can leak into another, causing flaky tests. Use the `@Sql` annotation to run a cleanup script after each test method to ensure a clean state.

**Good Example:**
```java
@SpringBootTest
@Sql(scripts = "/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
class MyServiceIntegrationTest {
    // ... test methods ...
}

// src/test/resources/cleanup.sql
// Use DELETE FROM to clear tables. TRUNCATE is faster but can have issues with foreign keys.
DELETE FROM my_table;
DELETE FROM another_table;
// Optional: Reset sequences for predictable IDs
ALTER SEQUENCE my_table_id_seq RESTART WITH 1;
```

**Bad Example:**
A `@SpringBootTest` class without any strategy for cleaning the database between tests. This will lead to tests that fail depending on the order they are run.


*Clarification*: @Sql scripts should reset sequences or auto-increment counters if predictable IDs are required (e.g., ALTER SEQUENCE my_table_id_seq RESTART WITH 1 for PostgreSQL).

*Tip*: Pair @Sql with @DirtiesContext to ensure a fresh application context after specific tests. Only on need basis and not by default

*Warning*: Avoid TRUNCATE in cleanup scripts if it violates foreign key constraints (unless you explicitly handle cascading or disable constraints for the test).

---

## Rule 3: Do Not Rely on Hardcoded IDs
**Title:** Never Assert Hardcoded IDs in Tests
**Description:** Never assume that an auto-generated ID will have a predictable value (e.g., 1, 2, 3). The order of test execution is not guaranteed. Always use the entity instance returned by the `save()` method to get the actual, generated ID for your assertions.

**Good Example:**
```java
@Test
void shouldSaveAndReturnEntityWithId() {
    MyEntity entity = new MyEntity("test");
    MyEntity savedEntity = repository.save(entity);

    // Good: Assert that an ID was generated.
    assertThat(savedEntity.getId()).isNotNull();

    // Good: Use the returned ID to fetch the entity again.
    Optional<MyEntity> foundEntity = repository.findById(savedEntity.getId());
    assertThat(foundEntity).isPresent();
}
```

**Bad Example:**
```java
@Test
void shouldSaveWithIdOne() {
    MyEntity entity = new MyEntity("test");
    repository.save(entity);

    // PROBLEM: This is a flaky test. It will fail if any other test
    // that runs before it also saves an entity to this table.
    assertThat(entity.getId()).isEqualTo(1L); 
}
```

*Reinforcement*: Hardcoded IDs are acceptable only in tightly controlled, isolated tests where the database is reset beforehand (e.g., via @Rollback or @Sql).
*Tip*: Use @Rollback(false) sparingly (e.g., for tests requiring post-test validation) but ensure cleanup is explicitly handled
