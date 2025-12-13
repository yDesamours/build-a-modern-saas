# Module 3.1: Directory Structure Best Practices

---

## 🎯 Learning Objectives

By the end of this section, you will:

- Understand best practices for organizing code in Java, Node.js, and Go
- Implement scalable directory structures for SaaS applications
- Know where to place different types of files (controllers, services, models)
- Apply layered architecture principles through folder organization
- Create consistent structure across your projects

---

## 📁 Java/Spring Boot Directory Structure

### Standard Maven/Gradle Structure

```
my-saas-app/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── myapp/
│   │   │           ├── MyApplication.java          # Main entry point
│   │   │           │
│   │   │           ├── config/                     # Configuration classes
│   │   │           │   ├── SecurityConfig.java
│   │   │           │   ├── DatabaseConfig.java
│   │   │           │   ├── CacheConfig.java
│   │   │           │   └── AsyncConfig.java
│   │   │           │
│   │   │           ├── controller/                 # REST controllers
│   │   │           │   ├── UserController.java
│   │   │           │   ├── OrderController.java
│   │   │           │   ├── ProductController.java
│   │   │           │   └── api/                    # API versioning
│   │   │           │       ├── v1/
│   │   │           │       │   ├── UserControllerV1.java
│   │   │           │       │   └── OrderControllerV1.java
│   │   │           │       └── v2/
│   │   │           │           └── UserControllerV2.java
│   │   │           │
│   │   │           ├── service/                    # Business logic layer
│   │   │           │   ├── UserService.java
│   │   │           │   ├── OrderService.java
│   │   │           │   ├── ProductService.java
│   │   │           │   └── impl/                   # Service implementations
│   │   │           │       ├── UserServiceImpl.java
│   │   │           │       └── OrderServiceImpl.java
│   │   │           │
│   │   │           ├── repository/                 # Data access layer
│   │   │           │   ├── UserRepository.java
│   │   │           │   ├── OrderRepository.java
│   │   │           │   └── ProductRepository.java
│   │   │           │
│   │   │           ├── model/                      # Domain models/entities
│   │   │           │   ├── entity/                 # JPA entities
│   │   │           │   │   ├── User.java
│   │   │           │   │   ├── Order.java
│   │   │           │   │   └── Product.java
│   │   │           │   └── enums/                  # Enumerations
│   │   │           │       ├── UserRole.java
│   │   │           │       └── OrderStatus.java
│   │   │           │
│   │   │           ├── dto/                        # Data Transfer Objects
│   │   │           │   ├── request/                # Request DTOs
│   │   │           │   │   ├── CreateUserRequest.java
│   │   │           │   │   └── UpdateOrderRequest.java
│   │   │           │   └── response/               # Response DTOs
│   │   │           │       ├── UserResponse.java
│   │   │           │       └── OrderResponse.java
│   │   │           │
│   │   │           ├── mapper/                     # Entity <-> DTO mappers
│   │   │           │   ├── UserMapper.java
│   │   │           │   └── OrderMapper.java
│   │   │           │
│   │   │           ├── exception/                  # Custom exceptions
│   │   │           │   ├── GlobalExceptionHandler.java
│   │   │           │   ├── ResourceNotFoundException.java
│   │   │           │   ├── BadRequestException.java
│   │   │           │   └── UnauthorizedException.java
│   │   │           │
│   │   │           ├── security/                   # Security components
│   │   │           │   ├── JwtTokenProvider.java
│   │   │           │   ├── JwtAuthenticationFilter.java
│   │   │           │   └── UserDetailsServiceImpl.java
│   │   │           │
│   │   │           ├── validation/                 # Custom validators
│   │   │           │   ├── EmailValidator.java
│   │   │           │   └── PhoneValidator.java
│   │   │           │
│   │   │           └── util/                       # Utility classes
│   │   │               ├── DateUtils.java
│   │   │               ├── StringUtils.java
│   │   │               └── Constants.java
│   │   │
│   │   └── resources/
│   │       ├── application.yml                     # Main configuration
│   │       ├── application-dev.yml                 # Dev environment
│   │       ├── application-prod.yml                # Production environment
│   │       ├── application-test.yml                # Test environment
│   │       ├── db/
│   │       │   └── migration/                      # Flyway/Liquibase migrations
│   │       │       ├── V1__create_users_table.sql
│   │       │       ├── V2__create_orders_table.sql
│   │       │       └── V3__add_user_email_index.sql
│   │       ├── static/                             # Static resources
│   │       │   ├── css/
│   │       │   ├── js/
│   │       │   └── images/
│   │       └── templates/                          # Email templates
│   │           ├── welcome-email.html
│   │           └── password-reset.html
│   │
│   └── test/
│       ├── java/
│       │   └── com/
│       │       └── myapp/
│       │           ├── controller/                 # Controller tests
│       │           │   ├── UserControllerTest.java
│       │           │   └── OrderControllerTest.java
│       │           ├── service/                    # Service tests
│       │           │   ├── UserServiceTest.java
│       │           │   └── OrderServiceTest.java
│       │           ├── repository/                 # Repository tests
│       │           │   └── UserRepositoryTest.java
│       │           └── integration/                # Integration tests
│       │               ├── UserIntegrationTest.java
│       │               └── OrderIntegrationTest.java
│       │
│       └── resources/
│           ├── application-test.yml
│           └── test-data.sql
│
├── build.gradle (or pom.xml)                      # Build configuration
├── gradle.properties                               # Gradle properties
├── settings.gradle                                 # Project settings
├── .gitignore
├── README.md
└── Dockerfile
```

### Domain-Driven Design (DDD) Structure

For more complex applications, consider DDD approach:

```
my-saas-app/
├── src/
│   └── main/
│       └── java/
│           └── com/
│               └── myapp/
│                   ├── Application.java
│                   │
│                   ├── domain/                     # Domain layer
│                   │   ├── user/                   # User bounded context
│                   │   │   ├── model/
│                   │   │   │   ├── User.java           # Aggregate root
│                   │   │   │   ├── UserProfile.java    # Entity
│                   │   │   │   └── Email.java          # Value object
│                   │   │   ├── repository/
│                   │   │   │   └── UserRepository.java
│                   │   │   ├── service/
│                   │   │   │   └── UserDomainService.java
│                   │   │   └── event/
│                   │   │       └── UserCreatedEvent.java
│                   │   │
│                   │   ├── order/                  # Order bounded context
│                   │   │   ├── model/
│                   │   │   │   ├── Order.java
│                   │   │   │   ├── OrderItem.java
│                   │   │   │   └── Money.java
│                   │   │   ├── repository/
│                   │   │   │   └── OrderRepository.java
│                   │   │   └── service/
│                   │   │       └── OrderDomainService.java
│                   │   │
│                   │   └── shared/                 # Shared kernel
│                   │       ├── valueobject/
│                   │       │   ├── Money.java
│                   │       │   └── Address.java
│                   │       └── event/
│                   │           └── DomainEvent.java
│                   │
│                   ├── application/                # Application layer
│                   │   ├── user/
│                   │   │   ├── UserApplicationService.java
│                   │   │   ├── dto/
│                   │   │   │   ├── CreateUserCommand.java
│                   │   │   │   └── UserDTO.java
│                   │   │   └── mapper/
│                   │   │       └── UserMapper.java
│                   │   └── order/
│                   │       └── OrderApplicationService.java
│                   │
│                   ├── infrastructure/             # Infrastructure layer
│                   │   ├── persistence/
│                   │   │   ├── jpa/
│                   │   │   │   ├── UserRepositoryImpl.java
│                   │   │   │   └── OrderRepositoryImpl.java
│                   │   │   └── entity/
│                   │   │       ├── UserEntity.java
│                   │   │       └── OrderEntity.java
│                   │   ├── messaging/
│                   │   │   └── RabbitMQEventPublisher.java
│                   │   └── external/
│                   │       └── StripePaymentGateway.java
│                   │
│                   └── interface/                  # Interface/Presentation layer
│                       ├── rest/
│                       │   ├── UserController.java
│                       │   └── OrderController.java
│                       └── graphql/
│                           └── UserGraphQLResolver.java
```

### Key Principles for Java Structure:

1. **Package by Feature, not by Layer**: Group related classes together
2. **Separate Entities and DTOs**: Never expose entities directly
3. **Use Interfaces**: Define service interfaces, implement separately
4. **Keep Controllers Thin**: Logic belongs in services
5. **Clear Layer Separation**: Controller → Service → Repository
6. **Organize by Bounded Context**: For DDD, group by domain concepts

---

## 📁 Node.js/TypeScript Directory Structure

### Express/NestJS Structure

```
my-saas-app/
├── src/
│   ├── main.ts                                     # Application entry point
│   │
│   ├── config/                                     # Configuration
│   │   ├── database.config.ts
│   │   ├── redis.config.ts
│   │   ├── jwt.config.ts
│   │   └── index.ts
│   │
│   ├── modules/                                    # Feature modules
│   │   ├── users/
│   │   │   ├── users.module.ts
│   │   │   ├── users.controller.ts
│   │   │   ├── users.service.ts
│   │   │   ├── users.repository.ts
│   │   │   ├── dto/
│   │   │   │   ├── create-user.dto.ts
│   │   │   │   ├── update-user.dto.ts
│   │   │   │   └── user-response.dto.ts
│   │   │   ├── entities/
│   │   │   │   └── user.entity.ts
│   │   │   ├── interfaces/
│   │   │   │   ├── user.interface.ts
│   │   │   │   └── users-service.interface.ts
│   │   │   ├── guards/
│   │   │   │   └── user-owner.guard.ts
│   │   │   └── tests/
│   │   │       ├── users.controller.spec.ts
│   │   │       └── users.service.spec.ts
│   │   │
│   │   ├── orders/
│   │   │   ├── orders.module.ts
│   │   │   ├── orders.controller.ts
│   │   │   ├── orders.service.ts
│   │   │   ├── dto/
│   │   │   ├── entities/
│   │   │   └── tests/
│   │   │
│   │   ├── products/
│   │   │   └── ... (similar structure)
│   │   │
│   │   └── auth/
│   │       ├── auth.module.ts
│   │       ├── auth.controller.ts
│   │       ├── auth.service.ts
│   │       ├── strategies/
│   │       │   ├── jwt.strategy.ts
│   │       │   └── local.strategy.ts
│   │       ├── guards/
│   │       │   ├── jwt-auth.guard.ts
│   │       │   └── roles.guard.ts
│   │       └── decorators/
│   │           └── roles.decorator.ts
│   │
│   ├── common/                                     # Shared across modules
│   │   ├── decorators/                             # Custom decorators
│   │   │   ├── api-paginated-response.decorator.ts
│   │   │   └── current-user.decorator.ts
│   │   ├── filters/                                # Exception filters
│   │   │   ├── http-exception.filter.ts
│   │   │   └── all-exceptions.filter.ts
│   │   ├── guards/                                 # Global guards
│   │   │   └── throttler.guard.ts
│   │   ├── interceptors/                           # Interceptors
│   │   │   ├── logging.interceptor.ts
│   │   │   ├── transform.interceptor.ts
│   │   │   └── timeout.interceptor.ts
│   │   ├── pipes/                                  # Validation pipes
│   │   │   └── validation.pipe.ts
│   │   ├── middleware/                             # Custom middleware
│   │   │   ├── logger.middleware.ts
│   │   │   └── cors.middleware.ts
│   │   ├── interfaces/                             # Common interfaces
│   │   │   ├── pagination.interface.ts
│   │   │   └── response.interface.ts
│   │   └── utils/                                  # Utility functions
│   │       ├── date.utils.ts
│   │       ├── string.utils.ts
│   │       └── crypto.utils.ts
│   │
│   ├── database/                                   # Database related
│   │   ├── migrations/                             # TypeORM migrations
│   │   │   ├── 1234567890-CreateUsersTable.ts
│   │   │   └── 1234567891-CreateOrdersTable.ts
│   │   ├── seeds/                                  # Database seeds
│   │   │   └── users.seed.ts
│   │   └── data-source.ts                          # TypeORM config
│   │
│   ├── events/                                     # Event handlers
│   │   ├── user-created.event.ts
│   │   ├── order-placed.event.ts
│   │   └── handlers/
│   │       ├── user-created.handler.ts
│   │       └── order-placed.handler.ts
│   │
│   ├── jobs/                                       # Background jobs
│   │   ├── email.job.ts
│   │   ├── report.job.ts
│   │   └── processors/
│   │       ├── email.processor.ts
│   │       └── report.processor.ts
│   │
│   └── types/                                      # TypeScript type definitions
│       ├── express.d.ts                            # Extend Express types
│       ├── environment.d.ts
│       └── global.d.ts
│
├── test/                                           # E2E tests
│   ├── app.e2e-spec.ts
│   ├── users.e2e-spec.ts
│   └── orders.e2e-spec.ts
│
├── .env                                            # Environment variables
├── .env.example                                    # Example env file
├── .eslintrc.js                                    # ESLint config
├── .prettierrc                                     # Prettier config
├── tsconfig.json                                   # TypeScript config
├── tsconfig.build.json                             # Build config
├── package.json
├── nest-cli.json                                   # NestJS CLI config
├── README.md
└── Dockerfile
```

### Clean Architecture Structure (Alternative)

```
my-saas-app/
├── src/
│   ├── main.ts
│   │
│   ├── core/                                       # Core business logic
│   │   ├── domain/                                 # Domain models
│   │   │   ├── user/
│   │   │   │   ├── user.entity.ts
│   │   │   │   ├── user-profile.value-object.ts
│   │   │   │   └── email.value-object.ts
│   │   │   └── order/
│   │   │       ├── order.entity.ts
│   │   │       └── order-item.value-object.ts
│   │   │
│   │   ├── usecases/                               # Use cases (application logic)
│   │   │   ├── user/
│   │   │   │   ├── create-user.usecase.ts
│   │   │   │   ├── get-user.usecase.ts
│   │   │   │   └── update-user.usecase.ts
│   │   │   └── order/
│   │   │       ├── create-order.usecase.ts
│   │   │       └── get-orders.usecase.ts
│   │   │
│   │   └── repositories/                           # Repository interfaces
│   │       ├── user.repository.interface.ts
│   │       └── order.repository.interface.ts
│   │
│   ├── infrastructure/                             # External concerns
│   │   ├── database/
│   │   │   ├── typeorm/
│   │   │   │   ├── entities/
│   │   │   │   │   ├── user.schema.ts
│   │   │   │   │   └── order.schema.ts
│   │   │   │   └── repositories/
│   │   │   │       ├── user.repository.ts
│   │   │   │       └── order.repository.ts
│   │   │   └── migrations/
│   │   ├── http/
│   │   │   └── axios/
│   │   │       └── http-client.service.ts
│   │   ├── messaging/
│   │   │   └── rabbitmq/
│   │   │       └── message-queue.service.ts
│   │   └── cache/
│   │       └── redis/
│   │           └── cache.service.ts
│   │
│   └── presentation/                               # Presentation layer
│       ├── rest/
│       │   ├── controllers/
│       │   │   ├── user.controller.ts
│       │   │   └── order.controller.ts
│       │   ├── dto/
│       │   ├── middleware/
│       │   └── validators/
│       └── graphql/
│           ├── resolvers/
│           └── schemas/
```

### Key Principles for Node.js Structure:

1. **Module-Based Organization**: Group by feature, not by file type
2. **Separation of Concerns**: Controller → Service → Repository
3. **Use Barrel Exports**: index.ts files for cleaner imports
4. **Keep DTOs Separate**: Never expose entities directly
5. **Centralize Common Code**: Shared utilities, guards, interceptors
6. **Type Everything**: Leverage TypeScript for type safety

---

## 📁 Go Directory Structure

### Standard Go Project Layout

```
my-saas-app/
├── cmd/                                            # Main applications
│   ├── api/                                        # API server
│   │   └── main.go
│   ├── worker/                                     # Background worker
│   │   └── main.go
│   └── migrate/                                    # Database migrations CLI
│       └── main.go
│
├── internal/                                       # Private application code
│   ├── api/                                        # API layer
│   │   ├── handler/                                # HTTP handlers
│   │   │   ├── user.go
│   │   │   ├── order.go
│   │   │   └── health.go
│   │   ├── middleware/                             # Middleware
│   │   │   ├── auth.go
│   │   │   ├── logging.go
│   │   │   ├── cors.go
│   │   │   └── ratelimit.go
│   │   ├── request/                                # Request DTOs
│   │   │   ├── user.go
│   │   │   └── order.go
│   │   ├── response/                               # Response DTOs
│   │   │   ├── user.go
│   │   │   └── order.go
│   │   └── router/                                 # Route setup
│   │       └── router.go
│   │
│   ├── domain/                                     # Business domain
│   │   ├── user/                                   # User domain
│   │   │   ├── user.go                             # User entity
│   │   │   ├── repository.go                       # Repository interface
│   │   │   ├── service.go                          # Business logic
│   │   │   └── errors.go                           # Domain errors
│   │   ├── order/                                  # Order domain
│   │   │   ├── order.go
│   │   │   ├── repository.go
│   │   │   ├── service.go
│   │   │   └── errors.go
│   │   └── shared/                                 # Shared domain
│   │       ├── valueobject/
│   │       │   ├── email.go
│   │       │   ├── money.go
│   │       │   └── address.go
│   │       └── event/
│   │           └── event.go
│   │
│   ├── repository/                                 # Data access implementations
│   │   ├── postgres/
│   │   │   ├── user.go                             # User repo implementation
│   │   │   ├── order.go                            # Order repo implementation
│   │   │   └── transaction.go
│   │   ├── redis/
│   │   │   └── cache.go
│   │   └── migrations/
│   │       ├── 000001_create_users_table.up.sql
│   │       ├── 000001_create_users_table.down.sql
│   │       ├── 000002_create_orders_table.up.sql
│   │       └── 000002_create_orders_table.down.sql
│   │
│   ├── service/                                    # Application services
│   │   ├── auth/
│   │   │   ├── jwt.go
│   │   │   ├── password.go
│   │   │   └── session.go
│   │   ├── email/
│   │   │   └── email.go
│   │   └── payment/
│   │       └── payment.go
│   │
│   └── infrastructure/                             # Infrastructure concerns
│       ├── database/
│       │   ├── postgres.go
│       │   └── redis.go
│       ├── queue/
│       │   └── rabbitmq.go
│       ├── cache/
│       │   └── cache.go
│       └── logger/
│           └── logger.go
│
├── pkg/                                            # Public libraries (reusable)
│   ├── httputil/                                   # HTTP utilities
│   │   ├── response.go
│   │   └── error.go
│   ├── validator/                                  # Validation utilities
│   │   └── validator.go
│   ├── jwt/                                        # JWT utilities
│   │   └── jwt.go
│   └── pagination/                                 # Pagination utilities
│       └── pagination.go
│
├── config/                                         # Configuration files
│   ├── config.go                                   # Config struct
│   ├── development.yaml
│   ├── production.yaml
│   └── test.yaml
│
├── scripts/                                        # Build/deployment scripts
│   ├── build.sh
│   ├── test.sh
│   └── deploy.sh
│
├── migrations/                                     # Database migrations (if not in internal)
│   └── ... (SQL files)
│
├── docs/                                           # Documentation
│   ├── api/                                        # API documentation
│   │   └── swagger.yaml
│   └── architecture/
│       └── diagrams/
│
├── test/                                           # Additional test files
│   ├── integration/                                # Integration tests
│   │   ├── user_test.go
│   │   └── order_test.go
│   └── testdata/                                   # Test fixtures
│       └── users.json
│
├── vendor/                                         # Vendored dependencies (optional)
│
├── .env                                            # Environment variables
├── .env.example
├── .gitignore
├── .golangci.yml                                   # Linter config
├── Makefile                                        # Build commands
├── go.mod                                          # Go modules
├── go.sum
├── Dockerfile
├── docker-compose.yml
└── README.md
```

### Hexagonal Architecture (Ports & Adapters) - Alternative

```
my-saas-app/
├── cmd/
│   └── api/
│       └── main.go
│
├── internal/
│   ├── core/                                       # Core domain (business logic)
│   │   ├── domain/                                 # Domain models
│   │   │   ├── user/
│   │   │   │   ├── user.go
│   │   │   │   └── user_test.go
│   │   │   └── order/
│   │   │       ├── order.go
│   │   │       └── order_test.go
│   │   │
│   │   ├── port/                                   # Ports (interfaces)
│   │   │   ├── repository/                         # Repository ports
│   │   │   │   ├── user.go
│   │   │   │   └── order.go
│   │   │   └── service/                            # Service ports
│   │   │       ├── email.go
│   │   │       └── payment.go
│   │   │
│   │   └── service/                                # Core services
│   │       ├── user_service.go
│   │       └── order_service.go
│   │
│   └── adapter/                                    # Adapters (implementations)
│       ├── http/                                   # HTTP adapter (primary)
│       │   ├── handler/
│       │   ├── middleware/
│       │   └── router/
│       ├── repository/                             # Repository adapters (secondary)
│       │   ├── postgres/
│       │   │   ├── user_repository.go
│       │   │   └── order_repository.go
│       │   └── mongodb/
│       │       └── user_repository.go
│       ├── email/                                  # Email adapter (secondary)
│       │   └── smtp/
│       │       └── email_service.go
│       └── payment/                                # Payment adapter (secondary)
│           ├── stripe/
│           │   └── payment_service.go
│           └── paypal/
│               └── payment_service.go
│
└── pkg/
    └── ... (shared utilities)
```

### Key Principles for Go Structure:

1. **cmd/ for Entry Points**: One directory per executable
2. **internal/ for Private Code**: Cannot be imported by other projects
3. **pkg/ for Public Libraries**: Can be imported by external projects
4. **Domain-Driven Packages**: Group by domain, not by layer
5. **Interface at Package Level**: Define interfaces where they're used
6. **Keep main.go Small**: Wire dependencies, start server
7. **Test Files Alongside Code**: user.go → user_test.go

---

## 🎯 Common Patterns Across All Stacks

### 1. Separate Concerns by Layer

```
✅ Good Structure:
api/
├── controller/     # Handles HTTP
├── service/        # Business logic
└── repository/     # Data access

❌ Bad Structure:
api/
└── everything.js   # All mixed together
```

### 2. Group by Feature, Not by Type

```
✅ Good Structure:
modules/
├── users/
│   ├── controller
│   ├── service
│   └── repository
└── orders/
    ├── controller
    ├── service
    └── repository

❌ Bad Structure:
controllers/
├── users
└── orders
services/
├── users
└── orders
```

### 3. Keep Configuration Separate

```
config/
├── database.config
├── redis.config
├── jwt.config
└── environment/
    ├── development
    ├── staging
    └── production
```

### 4. Organize Tests Alongside Code

```
users/
├── user.service.ts
├── user.service.spec.ts      # Unit test
├── user.controller.ts
└── user.controller.spec.ts    # Unit test

test/
└── users.e2e-spec.ts          # E2E test
```

### 5. Use Index/Barrel Files (Node.js/TS)

```typescript
// users/index.ts
export * from "./user.entity";
export * from "./user.service";
export * from "./user.controller";
export * from "./dto";

// Then import like:
import { UserService, CreateUserDto } from "./users";
```

---

## 📋 Directory Structure Checklist

### Essential Directories

- ✅ **Source code** (src/, internal/, cmd/)
- ✅ **Configuration** (config/, .env)
- ✅ **Tests** (test/, _\_test.go, _.spec.ts)
- ✅ **Documentation** (docs/, README.md)
- ✅ **Build artifacts** (.gitignore them!)
- ✅ **Database migrations** (migrations/, db/)
- ✅ **Static assets** (public/, static/)
- ✅ **Scripts** (scripts/, tools/)

### Anti-Patterns to Avoid

❌ **Deeply nested directories** (more than 4-5 levels)
❌ **God directories** (single directory with 50+ files)
❌ **Inconsistent naming** (users/ vs user/ vs User/)
❌ **Mixing concerns** (business logic in controllers)
❌ **No clear boundaries** (everything imports everything)
❌ **Circular dependencies** (A imports B, B imports A)
❌ **Exposing internal details** (returning entities from API)

---

## 🏆 Best Practices Summary

### For Java/Spring Boot

1. **Package by Feature**: `com.myapp.user` not `com.myapp.controller`
2. **Use DTOs**: Never expose entities in REST APIs
3. **Separate Interfaces**: Service interfaces in service package, implementations in impl/
4. **Resource Organization**: application.yml, static/, templates/ in resources/
5. **Test Structure**: Mirror main structure in test directory
6. **Use Annotations**: `@Controller`, `@Service`, `@Repository` in right places

### For Node.js/TypeScript

1. **Module-First**: Feature modules contain all related code
2. **Barrel Exports**: Use index.ts for clean imports
3. **Shared Common**: Common utilities, decorators, guards in common/
4. **Config Management**: Separate config directory with environment files
5. **Type Definitions**: Extend types in types/ directory
6. **NestJS Conventions**: Follow module/controller/service/repository pattern

### For Go

1. **cmd/ for Binaries**: Each executable gets its own directory
2. **internal/ for Private**: Application code that shouldn't be imported
3. **pkg/ for Public**: Reusable packages for external use
4. **Flat Packages**: Keep packages relatively flat
5. **Domain Packages**: Group by domain entities (user, order, product)
6. **Interface Placement**: Define interfaces where they're consumed
7. **Standard Project Layout**: Follow https://github.com/golang-standards/project-layout

---

## 🔍 Real-World Examples

### Microservice Structure Example (Node.js)

```
user-service/
├── src/
│   ├── main.ts
│   ├── modules/
│   │   └── users/
│   │       ├── users.module.ts
│   │       ├── users.controller.ts
│   │       ├── users.service.ts
│   │       ├── users.repository.ts
│   │       ├── dto/
│   │       ├── entities/
│   │       └── tests/
│   ├── common/
│   ├── config/
│   └── database/
├── test/
├── .env
├── package.json
└── Dockerfile

order-service/
├── src/
│   ├── main.ts
│   ├── modules/
│   │   └── orders/
│   │       ├── orders.module.ts
│   │       ├── orders.controller.ts
│   │       ├── orders.service.ts
│   │       └── ...
│   └── ...
└── ...
```

### Monolith to Microservices Migration (Java)

**Before (Monolith):**

```
my-app/
└── src/
    └── com/
        └── myapp/
            ├── user/
            ├── order/
            ├── product/
            └── payment/
```

**After (Bounded Contexts):**

```
user-context/
└── src/com/myapp/user/
    ├── domain/
    ├── application/
    └── infrastructure/

order-context/
└── src/com/myapp/order/
    ├── domain/
    ├── application/
    └── infrastructure/
```

---

## 📂 Folder Naming Conventions

### General Rules

| Type                   | Convention                | Example                 |
| ---------------------- | ------------------------- | ----------------------- |
| **Java Packages**      | lowercase, no underscores | `com.myapp.userservice` |
| **Java Classes**       | PascalCase                | `UserController.java`   |
| **TypeScript Files**   | kebab-case                | `user-controller.ts`    |
| **TypeScript Classes** | PascalCase                | `class UserController`  |
| **Go Packages**        | lowercase, no underscores | `userservice/`          |
| **Go Files**           | snake_case                | `user_repository.go`    |
| **Test Files (Go)**    | \*\_test.go               | `user_test.go`          |
| **Test Files (TS)**    | _.spec.ts or _.test.ts    | `user.spec.ts`          |
| **Test Files (Java)**  | \*Test.java               | `UserServiceTest.java`  |

### Directory Naming

- **Use plural for collections**: `users/`, `orders/`, not `user/`, `order/`
- **Use lowercase**: `config/`, not `Config/`
- **Be consistent**: Choose one style and stick to it
- **Descriptive names**: `authentication/` not `auth/` if clarity matters

---

## 🚀 Scalability Considerations

### Small Application (< 10 files)

```
simple-api/
├── main.js
├── config.js
├── routes.js
├── controllers/
├── models/
└── utils.js
```

### Medium Application (10-50 files)

```
my-api/
├── src/
│   ├── config/
│   ├── controllers/
│   ├── services/
│   ├── models/
│   ├── middleware/
│   └── utils/
├── test/
└── package.json
```

### Large Application (50+ files)

```
my-saas/
├── src/
│   ├── modules/          # Feature modules
│   │   ├── users/
│   │   ├── orders/
│   │   ├── products/
│   │   ├── payments/
│   │   └── notifications/
│   ├── common/           # Shared code
│   ├── config/
│   ├── database/
│   └── events/
├── test/
└── package.json
```

### Enterprise Application (100+ files)

```
my-enterprise-saas/
├── apps/                 # Multiple applications
│   ├── api/
│   ├── web/
│   └── admin/
├── libs/                 # Shared libraries
│   ├── shared-ui/
│   ├── shared-data/
│   └── shared-utils/
├── services/             # Microservices
│   ├── user-service/
│   ├── order-service/
│   └── payment-service/
└── tools/                # Build tools
```

---

## 🛠️ Migration Strategies

### Refactoring Flat Structure to Organized Structure

**Before:**

```
src/
├── user-controller.ts
├── user-service.ts
├── user-repository.ts
├── order-controller.ts
├── order-service.ts
├── order-repository.ts
└── ... (50 more files)
```

**After:**

```
src/
├── modules/
│   ├── users/
│   │   ├── user.controller.ts
│   │   ├── user.service.ts
│   │   └── user.repository.ts
│   └── orders/
│       ├── order.controller.ts
│       ├── order.service.ts
│       └── order.repository.ts
└── common/
```

**Migration Steps:**

1. **Create new structure**: Don't delete old files yet
2. **Move files incrementally**: One module at a time
3. **Update imports**: Fix all import paths
4. **Test thoroughly**: Ensure nothing breaks
5. **Update CI/CD**: Adjust build paths if needed
6. **Delete old structure**: Once everything works

### Script for Automated Migration (Example)

```bash
#!/bin/bash
# migrate-structure.sh

# Create new directory structure
mkdir -p src/modules/users
mkdir -p src/modules/orders
mkdir -p src/common

# Move user files
mv src/user-controller.ts src/modules/users/user.controller.ts
mv src/user-service.ts src/modules/users/user.service.ts
mv src/user-repository.ts src/modules/users/user.repository.ts

# Move order files
mv src/order-controller.ts src/modules/orders/order.controller.ts
mv src/order-service.ts src/modules/orders/order.service.ts
mv src/order-repository.ts src/modules/orders/order.repository.ts

# Update imports (using sed or similar)
find src -type f -name "*.ts" -exec sed -i 's|./user-controller|./modules/users/user.controller|g' {} +

echo "Migration complete! Test your application."
```

---

## 📊 Structure Decision Matrix

| Project Size            | Team Size  | Complexity   | Recommended Structure                   |
| ----------------------- | ---------- | ------------ | --------------------------------------- |
| Small (<10 files)       | 1-2 devs   | Simple       | Flat structure                          |
| Small (10-20 files)     | 2-5 devs   | Simple       | Basic layers (controller/service/model) |
| Medium (20-50 files)    | 5-10 devs  | Moderate     | Feature modules                         |
| Large (50-100 files)    | 10-20 devs | Complex      | Feature modules + DDD                   |
| Enterprise (100+ files) | 20+ devs   | Very Complex | Monorepo/Microservices + DDD            |

---

## 🎓 Learning Path

### Step 1: Start Simple

- Begin with basic layer separation
- Don't over-architect early

### Step 2: Introduce Modules

- Group related files together
- Create feature modules

### Step 3: Apply Patterns

- Implement Repository pattern
- Use DTOs for data transfer
- Apply clean architecture principles

### Step 4: Refactor Continuously

- Don't let structure decay
- Refactor as project grows
- Keep consistency

---

## 📚 Additional Resources

### Official Style Guides

- **Java**: [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- **TypeScript**: [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- **Go**: [Effective Go](https://go.dev/doc/effective_go)
- **Go Project Layout**: [Standard Go Project Layout](https://github.com/golang-standards/project-layout)

### Framework-Specific Guides

- **Spring Boot**: [Spring Boot Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- **NestJS**: [NestJS Documentation](https://docs.nestjs.com/)
- **Gin (Go)**: [Gin Web Framework](https://gin-gonic.com/docs/)

### Books

- **"Clean Architecture"** - Robert C. Martin
- **"Domain-Driven Design"** - Eric Evans
- **"Building Microservices"** - Sam Newman

---

## ✅ Quick Reference Checklist

### When Setting Up a New Project:

- [ ] Choose appropriate structure for project size
- [ ] Set up clear layer separation (controller/service/repository)
- [ ] Create feature-based modules
- [ ] Separate configuration from code
- [ ] Set up test directory mirroring source
- [ ] Create README with structure explanation
- [ ] Add .gitignore for build artifacts
- [ ] Set up CI/CD with correct build paths
- [ ] Document naming conventions
- [ ] Plan for growth (how will structure scale?)

### Regular Maintenance:

- [ ] Review structure monthly
- [ ] Refactor when adding 10+ new files
- [ ] Keep dependencies clean (no circular refs)
- [ ] Update documentation when structure changes
- [ ] Enforce conventions in code reviews
- [ ] Remove unused files/directories
- [ ] Keep module boundaries clear

---

## 🎯 Summary

**Key Takeaways:**

1. **Organization matters**: Good structure improves maintainability
2. **Start simple**: Don't over-architect early
3. **Be consistent**: Choose conventions and stick to them
4. **Group by feature**: Not by file type
5. **Separate concerns**: Clear layers (controller/service/repository)
6. **Plan for growth**: Structure should scale with project
7. **Follow conventions**: Use language/framework standards
8. **Document decisions**: Help future developers understand

**Remember:** The best structure is one that:

- ✅ Your team understands
- ✅ Makes code easy to find
- ✅ Scales with your project
- ✅ Follows community conventions
- ✅ Supports your architecture (layered, DDD, clean, etc.)

---

**Great work!** You now understand how to organize code effectively in all three tech stacks.
