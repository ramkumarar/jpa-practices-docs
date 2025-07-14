# Pagila Spring Boot REST API Structure - Package by Feature

## Root Package Structure
```
com.pagila.api/
├── config/
├── common/
└── features/
```

## 1. Film Management Feature
**Package:** `com.pagila.api.features.film`

### Repositories
- **FilmRepository** - CRUD operations for films, search by title, genre, rating
- **CategoryRepository** - Film categories management
- **LanguageRepository** - Language lookup operations
- **FilmActorRepository** - Many-to-many relationship between films and actors
- **FilmCategoryRepository** - Many-to-many relationship between films and categories

### Services
- **FilmService** - Business logic for film operations, search, filtering
- **CategoryService** - Category management and film categorization
- **LanguageService** - Language operations and film language assignment
- **FilmSearchService** - Full-text search operations using PostgreSQL's tsvector

### Controllers
- **FilmController** - CRUD operations for films, search endpoints
- **CategoryController** - Category management endpoints
- **LanguageController** - Language lookup endpoints

### DTOs
- **FilmDTO** - Film data transfer object
- **FilmSearchDTO** - Search criteria and results
- **CategoryDTO** - Category information
- **LanguageDTO** - Language information

## 2. Actor Management Feature
**Package:** `com.pagila.api.features.actor`

### Repositories
- **ActorRepository** - Actor CRUD operations, search by name
- **ActorFilmRepository** - Custom queries for actor-film relationships

### Services
- **ActorService** - Actor business logic, filmography management
- **ActorFilmService** - Managing actor-film associations

### Controllers
- **ActorController** - Actor CRUD and search endpoints
- **ActorFilmController** - Actor filmography endpoints

### DTOs
- **ActorDTO** - Actor information
- **ActorFilmographyDTO** - Actor with associated films
- **FilmCastDTO** - Film with cast information

## 3. Customer Management Feature
**Package:** `com.pagila.api.features.customer`

### Repositories
- **CustomerRepository** - Customer CRUD, search by name, email, store
- **AddressRepository** - Address management operations
- **CityRepository** - City lookup operations
- **CountryRepository** - Country lookup operations

### Services
- **CustomerService** - Customer business logic, registration, profile management
- **AddressService** - Address management and validation
- **LocationService** - Geographic location operations (country, city, address)

### Controllers
- **CustomerController** - Customer registration, profile management
- **AddressController** - Address management endpoints
- **LocationController** - Geographic lookup endpoints

### DTOs
- **CustomerDTO** - Customer information
- **CustomerRegistrationDTO** - Customer registration data
- **AddressDTO** - Address information
- **LocationDTO** - Combined country, city, address information

## 4. Rental Management Feature
**Package:** `com.pagila.api.features.rental`

### Repositories
- **RentalRepository** - Rental transactions, active rentals, overdue rentals
- **InventoryRepository** - Inventory management, availability checks
- **StoreRepository** - Store operations and inventory by store

### Services
- **RentalService** - Rental business logic, checkout, return operations
- **InventoryService** - Inventory management, availability checks
- **StoreService** - Store operations and management

### Controllers
- **RentalController** - Rental checkout, return, history endpoints
- **InventoryController** - Inventory status and management
- **StoreController** - Store information and inventory

### DTOs
- **RentalDTO** - Rental transaction information
- **RentalCheckoutDTO** - Rental checkout request
- **RentalReturnDTO** - Rental return request
- **InventoryDTO** - Inventory item information
- **StoreDTO** - Store information

## 5. Payment Management Feature
**Package:** `com.pagila.api.features.payment`

### Repositories
- **PaymentRepository** - Payment operations across partitioned tables
- **PaymentPartitionRepository** - Partition-specific operations
- **PaymentHistoryRepository** - Historical payment queries

### Services
- **PaymentService** - Payment processing, validation, history
- **PaymentPartitionService** - Partition management and routing
- **PaymentReportService** - Payment analytics and reporting

### Controllers
- **PaymentController** - Payment processing endpoints
- **PaymentHistoryController** - Payment history and reports
- **PaymentReportController** - Payment analytics endpoints

### DTOs
- **PaymentDTO** - Payment information
- **PaymentRequestDTO** - Payment processing request
- **PaymentHistoryDTO** - Payment history response
- **PaymentReportDTO** - Payment analytics data

## 6. Staff Management Feature
**Package:** `com.pagila.api.features.staff`

### Repositories
- **StaffRepository** - Staff CRUD operations, authentication
- **StaffStoreRepository** - Staff-store assignments

### Services
- **StaffService** - Staff management, authentication, authorization
- **StaffAuthenticationService** - Authentication and session management

### Controllers
- **StaffController** - Staff management endpoints
- **StaffAuthController** - Authentication endpoints

### DTOs
- **StaffDTO** - Staff information
- **StaffAuthenticationDTO** - Authentication request/response
- **StaffProfileDTO** - Staff profile information

## 7. Analytics and Reporting Feature
**Package:** `com.pagila.api.features.analytics`

### Repositories
- **AnalyticsRepository** - Custom queries for business analytics
- **ReportRepository** - Report generation queries

### Services
- **AnalyticsService** - Business intelligence operations
- **ReportService** - Report generation and caching
- **DashboardService** - Dashboard data aggregation

### Controllers
- **AnalyticsController** - Analytics endpoints
- **ReportController** - Report generation endpoints
- **DashboardController** - Dashboard data endpoints

### DTOs
- **AnalyticsDTO** - Analytics data
- **ReportDTO** - Report structure
- **DashboardDTO** - Dashboard metrics

## 8. Common/Shared Components

### Configuration (`com.pagila.api.config`)
- **DatabaseConfig** - Database connection and JPA configuration
- **SecurityConfig** - Spring Security configuration
- **WebConfig** - Web MVC configuration
- **CacheConfig** - Redis/Cache configuration

### Common (`com.pagila.api.common`)

#### Exception Handling
- **GlobalExceptionHandler** - Global exception handling
- **PagilaException** - Custom exception classes
- **ErrorResponse** - Standardized error response format

#### Utilities
- **PaginationUtil** - Pagination helper utilities
- **ValidationUtil** - Custom validation utilities
- **DateUtil** - Date/time utilities
- **SearchUtil** - Search query building utilities

#### Base Classes
- **BaseEntity** - Common entity fields (id, timestamps)
- **BaseRepository** - Common repository operations
- **BaseService** - Common service operations
- **BaseController** - Common controller operations

#### DTOs
- **PagedResponse** - Standardized pagination response
- **SearchCriteria** - Common search criteria
- **ApiResponse** - Standardized API response wrapper

## 9. Security and Authentication
**Package:** `com.pagila.api.security`

### Components
- **JwtAuthenticationFilter** - JWT token validation
- **AuthenticationService** - Authentication logic
- **AuthorizationService** - Role-based authorization
- **SecurityUtil** - Security utilities

## 10. Integration and External Services
**Package:** `com.pagila.api.integration`

### Services
- **NotificationService** - Email/SMS notifications
- **PaymentGatewayService** - External payment processing
- **AuditService** - Audit logging and tracking

## Key Features and Considerations

### Cross-Cutting Concerns
- **Validation**: Bean validation on all DTOs
- **Caching**: Redis caching for frequently accessed data
- **Pagination**: Consistent pagination across all endpoints
- **Search**: Full-text search capabilities
- **Audit**: Comprehensive audit logging
- **Security**: JWT-based authentication and role-based authorization

### API Design Patterns
- **RESTful URLs**: Following REST conventions
- **HATEOAS**: Hypermedia links where appropriate
- **Versioning**: API versioning strategy
- **Rate Limiting**: API rate limiting and throttling
- **Documentation**: OpenAPI/Swagger documentation

### Performance Optimizations
- **Connection Pooling**: Database connection pooling
- **Query Optimization**: Efficient JPA queries
- **Lazy Loading**: Proper entity relationship loading
- **Caching Strategy**: Multi-level caching (L1, L2, Redis)
- **Partition Handling**: Efficient partition table operations

This structure provides a clean, maintainable, and scalable API architecture that follows Spring Boot best practices while leveraging the full capabilities of the Pagila database schema.