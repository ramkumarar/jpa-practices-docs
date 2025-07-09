---
description: "Expert-level rules for configuring optimal JPA and Hibernate settings for a PostgreSQL database in Spring Boot."
alwaysApply: true
---
# Expert Guide: Optimal PostgreSQL and JPA Configuration

This guide provides a production-ready, performance-oriented configuration for a Spring Boot application using PostgreSQL, JPA, and Hibernate. The settings are based on deep experience with the entire persistence stack.

## Rule 1: Establish a Robust Production Configuration
**Title:** Use a Battle-Tested, Performance-Oriented Configuration for Production
**Description:** The following properties represent a comprehensive and expert-recommended baseline for any production application. They are organized logically and include explanations for why each setting is critical for performance, stability, and correctness when using PostgreSQL.

**`application.properties` (Production Baseline)**
```properties
# ===================================================================================
# I. DATASOURCE CONFIGURATION (HikariCP & PostgreSQL JDBC)
#
# Expert Tip: The interaction between the connection pool (HikariCP) and the
# JDBC driver is critical for performance and stability.
# ===================================================================================

# The JDBC URL is the most critical piece for PostgreSQL performance.
# - reWriteBatchedInserts=true: This is NON-NEGOTIABLE. It unlocks true JDBC batch
#   inserts for PostgreSQL, dramatically improving bulk insert performance.
# - sslmode=require: Always use SSL for production database connections.
spring.datasource.url=jdbc:postgresql://your-db-host:5432/your_db?reWriteBatchedInserts=true&sslmode=require
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name=org.postgresql.Driver

# --- HikariCP Connection Pool Tuning ---
# Expert Tip: A well-tuned pool is key to application responsiveness under load.
spring.datasource.hikari.pool-name=Main-Hikari-Pool
spring.datasource.hikari.auto-commit=false
spring.datasource.hikari.connection-timeout=30000 # 30 seconds
spring.datasource.hikari.max-lifetime=900000      # 15 minutes. Prevents stale connections behind firewalls.
spring.datasource.hikari.maximum-pool-size=20     # Start with a reasonable size. Test and tune based on load.
spring.datasource.hikari.minimum-idle=10          # Keep a warm pool of connections ready for traffic spikes.

# --- JDBC Driver-Level Performance Tuning (Statement Caching) ---
# Expert Tip: Caching prepared statements avoids expensive parsing by the database.
spring.datasource.hikari.data-source-properties.cachePrepStmts=true
spring.datasource.hikari.data-source-properties.prepStmtCacheSize=250
spring.datasource.hikari.data-source-properties.prepStmtCacheSqlLimit=2048

# ===================================================================================
# II. JPA & HIBERNATE CORE CONFIGURATION
#
# Expert Tip: Correct Hibernate configuration prevents common and severe issues.
# ===================================================================================

# CRITICAL: Disable the "Open Session in View" anti-pattern. This prevents holding
# database connections for the entire web request, a common cause of pool exhaustion.
spring.jpa.open-in-view=false

# Use 'validate' to ensure your entities match the DB schema on startup without
# making changes. Use a dedicated migration tool (Flyway, Liquibase) for schema management.
# 'none' is also safe for production. NEVER use 'update' or 'create' in production.
spring.jpa.hibernate.ddl-auto=validate

# Tell Hibernate the pool provides non-auto-committing connections. This enables
# lazy connection acquisition, a significant performance boost.
spring.jpa.properties.hibernate.connection.provider_disables_autocommit=true

# Standardize on UTC for all timestamp operations to avoid timezone-related bugs.
spring.jpa.properties.hibernate.jdbc.time_zone=UTC

# ===================================================================================
# III. HIBERNATE PERFORMANCE & BATCHING
#
# Expert Tip: Batching is the single most important feature for bulk data operations.
# ===================================================================================

# Set the number of DML operations to group together in a single database roundtrip.
spring.jpa.properties.hibernate.jdbc.batch_size=50

# CRITICAL for batching: Allows Hibernate to reorder DML statements to create
# larger, more effective batches.
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true

# ===================================================================================
# IV. HIBERNATE QUERY OPTIMIZATIONS & SAFEGUARDS
#
# Expert Tip: These settings prevent common pitfalls and optimize query execution.
# ===================================================================================

# Fail fast if you attempt to paginate a query that fetches a collection in memory,
# which can cause OutOfMemoryError. This is a vital safeguard.
spring.jpa.properties.hibernate.query.fail_on_pagination_over_collection_fetch=true

# Improves statement cache efficiency for queries using 'IN' clauses by padding
# the parameter list, reducing the number of unique SQL strings generated.
spring.jpa.properties.hibernate.query.in_clause_parameter_padding=true

# Increase the size of Hibernate's cache for parsed HQL/JPQL query plans.
# A must-have for applications with a large number of distinct queries.
spring.jpa.properties.hibernate.query.plan_cache_max_size=2048

# Additional Configuration Properties to Consider:
# hibernate.dialect:  Specifies the database dialect (e.g., org.hibernate.dialect.PostgreSQLDialect). Crucial for correct SQL generation.
```

---

## Rule 2: Use Diagnostic Properties for Development
**Title:** Augment Configuration with Diagnostic Properties in Development Profiles
**Description:** In development and testing (`application-dev.properties`), enable detailed logging and statistics to debug issues and verify performance characteristics like batching and cache efficiency. **These settings must be disabled in production due to their performance overhead.**

**`application-dev.properties` (Development/Testing additions)**
```properties
# ===================================================================================
# V. DEVELOPMENT & DEBUGGING PROPERTIES (DO NOT USE IN PRODUCTION)
# ===================================================================================

# Disable the basic 'show-sql' in favor of more powerful logging below.
spring.jpa.show-sql=false

# Enable Hibernate's statistics-gathering mechanism.
spring.jpa.properties.hibernate.generate_statistics=true

# --- Fine-Grained Logging Configuration ---
# Expert Tip: Use a datasource-proxy for readable, formatted SQL logs.
# If not using a proxy, the org.hibernate logs are the next best thing.

# Log the actual SQL statements sent to the database.
logging.level.org.hibernate.SQL=DEBUG

# Log the parameter values bound to JDBC statements. Invaluable for debugging.
logging.level.org.hibernate.type.descriptor.sql=TRACE

# Log Hibernate's own performance statistics (cache hits/misses, batch counts, etc.).
# This is how you VERIFY that your batching configuration is working correctly.
logging.level.org.hibernate.stat=INFO
```
