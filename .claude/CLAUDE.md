# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build and Run Commands

### Build Solution
```bash
dotnet build src/eshop-microservices.sln
```

### Run All Services with Docker Compose
```bash
cd src
docker-compose up -d
```

### Run Individual Services (Development)
Each service runs on specific ports:

```bash
# Catalog Service (ports 6000/6060)
cd src/Services/Catalog/Catalog.API
dotnet run

# Basket Service (ports 6001/6061)
cd src/Services/Basket/Basket.API
dotnet run

# Discount Service - gRPC (ports 6002/6062)
cd src/Services/Discount/Discount.Grpc
dotnet run

# Ordering Service (ports 6003/6063)
cd src/Services/Ordering/Ordering.API
dotnet run
```

### Database Migrations (Ordering Service)
```bash
cd src/Services/Ordering/Ordering.API
dotnet ef migrations add <MigrationName>
dotnet ef database update
```

### Testing
```bash
# Run all tests
dotnet test src/eshop-microservices.sln

# Run tests for specific project
dotnet test src/Services/Catalog/Catalog.API.Tests
```

## Architecture Overview

This is a **microservices-based e-commerce platform** built with .NET 9.0, demonstrating production-ready patterns including CQRS, DDD, and event-driven architecture.

### Microservices Structure

**4 Independent Services**, each with its own database (Database per Service pattern):

1. **Catalog.API**: Product catalog management (PostgreSQL + Marten)
2. **Basket.API**: Shopping cart with caching (PostgreSQL + Marten + Redis)
3. **Discount.Grpc**: Discount/coupon service (SQLite, gRPC protocol)
4. **Ordering.API**: Order processing with Clean Architecture (SQL Server + EF Core)

### Inter-Service Communication

- **gRPC**: Basket service calls Discount service synchronously via gRPC to apply discounts
- **Binary Protocol**: gRPC uses Protocol Buffers for high-performance communication
- **Service Discovery**: Services communicate via configured URLs in `appsettings.json` or docker-compose

### BuildingBlocks (Shared Library)

Located at `src/BuildingBlocks/BuildingBlocks/`, this shared library contains:

- **CQRS Interfaces**: `ICommand<TResponse>`, `IQuery<TResponse>`, `ICommandHandler`, `IQueryHandler`
- **MediatR Behaviors**: `ValidationBehavior` (FluentValidation integration), `LoggingBehavior`
- **Common Patterns**: Exceptions, pagination support
- **Key Insight**: All CQRS interfaces extend MediatR's `IRequest<TResponse>` for pipeline integration

## Key Architectural Patterns

### CQRS with MediatR

All services use CQRS pattern with MediatR for command/query separation:

```csharp
// Command example
public record CreateOrderCommand(OrderDto Order) : ICommand<CreateOrderResult>;
public class CreateOrderHandler : ICommandHandler<CreateOrderCommand, CreateOrderResult> { }

// Query example
public record GetOrdersQuery(PaginationRequest Request) : IQuery<GetOrdersResult>;
public class GetOrdersHandler : IQueryHandler<GetOrdersQuery, GetOrdersResult> { }
```

**Pipeline Behaviors** (automatic for all commands/queries):
1. `ValidationBehavior`: Executes FluentValidation validators before handler
2. `LoggingBehavior`: Logs request/response with timing

### Ordering Service - Clean Architecture (4-Layer DDD)

The Ordering service follows strict Clean Architecture with clear layer separation:

**Ordering.Domain** (innermost layer):
- Aggregates: `Order` (aggregate root with domain events)
- Entities: `Customer`, `OrderItem`, `Product`
- Value Objects: `OrderId`, `CustomerId`, `OrderName`, `Address`, `Payment`
- Abstractions: `Aggregate<TId>`, `Entity<TId>`, `IDomainEvent`, `IAggregate`
- Key: Rich domain models with business logic, no dependencies

**Ordering.Application**:
- Commands/Queries with handlers (CQRS)
- DTOs, mapping (Mapster)
- `DependencyInjection.cs`: Registers MediatR with behaviors
- Depends on: Domain layer only

**Ordering.Infrastructure**:
- EF Core DbContext, configurations, interceptors
- Data access implementations
- Database initialization/seeding
- Depends on: Domain + Application

**Ordering.API**:
- Carter endpoints (minimal APIs)
- Dependency injection setup via extension methods
- Depends on: All inner layers

**Layer Communication**: API → Application → Domain ← Infrastructure

### Decorator Pattern (Basket Service)

The Basket service demonstrates the Decorator pattern for transparent caching:

```csharp
// Registration in Program.cs
builder.Services.AddScoped<IBasketRepository, BasketRepository>();
builder.Services.Decorate<IBasketRepository, CachedBasketRepository>();

// CachedBasketRepository wraps BasketRepository
// Checks Redis cache first, falls back to PostgreSQL, updates cache
```

### Minimal APIs with Carter

All services use Carter for clean endpoint organization:
- Each endpoint is a separate class implementing `ICarterModule`
- Routes defined in `AddRoutes(IEndpointRouteBuilder app)`
- Automatically discovered and registered via `app.MapCarter()`

### Marten for Event Sourcing

Catalog and Basket services use Marten as a document database over PostgreSQL:
- Lightweight sessions for better performance
- Schema configuration: `opts.Schema.For<Type>().Identity(x => x.Property)`
- No explicit migrations needed

### Domain Events (Ordering)

The `Aggregate<TId>` base class supports domain events:
- Aggregates collect domain events (`AddDomainEvent`)
- Events cleared after processing (`ClearDomainEvents`)
- Used for cross-aggregate communication and eventual consistency

## Service-Specific Details

### Catalog.API
- **Database**: PostgreSQL (port 5432) with Marten
- **Pattern**: Simple CQRS without complex DDD
- **Model**: `Product` with properties: Id, Name, Category, Description, ImageFile, Price

### Basket.API
- **Database**: PostgreSQL (port 5433) with Marten
- **Cache**: Redis (port 6379) via `StackExchange.Redis`
- **gRPC Client**: Calls `DiscountProtoService` to apply discounts
- **Decorator**: `CachedBasketRepository` wraps `BasketRepository`
- **Model**: `ShoppingCart` with `UserName` as identity

### Discount.Grpc
- **Database**: SQLite (file-based, `discountdb`)
- **Protocol**: gRPC only (no REST)
- **Model**: `CouponModel` (Id, ProductName, Description, Amount)
- **Note**: Certificate validation disabled in dev (`DangerousAcceptAnyServerCertificateValidator`)

### Ordering.API
- **Database**: SQL Server (port 1434)
- **ORM**: Entity Framework Core 9.0
- **Password**: SwN12345678 (SA account in docker-compose)
- **Initialization**: `InitialiseDatabaseAsync()` called in Development environment
- **Architecture**: Strict 4-layer Clean Architecture with DDD

## Configuration

### Connection Strings

Defined in `appsettings.json` and overridden in docker-compose:

- **Catalog**: `Host=catalogdb;Port=5432;Database=CatalogDb;Username=postgres;Password=postgres`
- **Basket**: `Server=basketdb;Port=5432;Database=BasketDb;User Id=postgres;Password=postgres`
- **Redis**: `distributedcache:6379`
- **Discount**: `Data Source=discountdb` (SQLite)
- **Ordering**: SQL Server on orderdb:1433

### gRPC Settings

Basket service connects to Discount via:
```json
"GrpcSettings": {
  "DiscountUrl": "https://discount.grpc:8081"
}
```

## Docker Infrastructure

Services defined in `src/docker-compose.yml`:
- **catalogdb**: postgres:17 (port 5432)
- **basketdb**: postgres:17 (port 5433)
- **distributedcache**: redis (port 6379)
- **orderdb**: mssql/server (port 1434)

All services expose both HTTP (8080) and HTTPS (8081) ports internally.

## Health Checks

All services implement health checks at `/health`:
- Database connectivity checks (PostgreSQL, SQL Server, Redis)
- Returns detailed status via `HealthChecks.UI.Client`

## Development Workflow

1. **Start infrastructure**: `docker-compose up -d` (databases + redis)
2. **Run migrations** (Ordering only): `dotnet ef database update`
3. **Start services** individually or via docker-compose
4. **Access endpoints**: Services on ports 6000-6003 (HTTP) and 6060-6063 (HTTPS)

## Adding New Features

### New Endpoint in Existing Service
1. Create a new Carter module implementing `ICarterModule`
2. Define command/query record with corresponding handler
3. Add FluentValidation validator if needed (auto-validated by pipeline)
4. Endpoint automatically registered via Carter

### New Command/Query
1. Create record implementing `ICommand<TResponse>` or `IQuery<TResponse>`
2. Create handler implementing `ICommandHandler` or `IQueryHandler`
3. MediatR automatically discovers handlers via assembly scanning

### New Microservice
1. Follow Database per Service pattern
2. Add to docker-compose for infrastructure
3. Reference BuildingBlocks for shared patterns
4. Implement health checks for monitoring

## Important Notes

- **No Shared Domain Models**: Each service owns its data models (bounded contexts)
- **Async/Await**: All operations use async patterns for scalability
- **Nullable Reference Types**: Enabled project-wide (`<Nullable>enable</Nullable>`)
- **Exception Handling**: Global exception handler via `CustomExceptionHandler`
- **Certificate Validation**: Disabled in dev for gRPC (`DangerousAcceptAnyServerCertificateValidator`)
