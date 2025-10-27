# 🛒 EShop Microservices Platform

[![.NET](https://img.shields.io/badge/.NET-9.0-512BD4?logo=dotnet)](https://dotnet.microsoft.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-DC382D?logo=redis&logoColor=white)](https://redis.io/)
[![gRPC](https://img.shields.io/badge/gRPC-244c5a?logo=google&logoColor=white)](https://grpc.io/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)

A modern, production-ready e-commerce platform built with .NET 9.0 microservices architecture, demonstrating industry best practices including Domain-Driven Design (DDD), CQRS, Event Sourcing, and inter-service communication patterns.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Services](#services)
- [Technologies](#technologies)
- [Key Patterns](#key-patterns)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Database Strategy](#database-strategy)

## 🎯 Overview

EShop is a microservices-based e-commerce platform that showcases:

- **Scalable Architecture**: Independent, containerized services that can scale horizontally
- **Modern .NET Stack**: Built on .NET 9.0 with the latest features and performance improvements
- **Domain-Driven Design**: Rich domain models with proper separation of concerns
- **Event-Driven Communication**: Services communicate through events and gRPC
- **Production-Ready Patterns**: CQRS, Repository, Decorator, and more

## 🏗️ Architecture

The platform follows a **microservices architecture** with independent services that own their data and business logic. Each service can be deployed, scaled, and maintained independently.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Catalog   │     │   Basket    │────▶│  Discount   │     │  Ordering   │
│   Service   │     │   Service   │gRPC │   Service   │     │   Service   │
└──────┬──────┘     └──────┬──────┘     └─────────────┘     └──────┬──────┘
       │                   │                                         │
       ▼                   ▼                                         ▼
  PostgreSQL          PostgreSQL                                SQL Server
   (Marten)           (Marten)                                   (EF Core)
                          │
                          ▼
                       Redis
                      (Cache)
```

### Core Principles

- **Database per Service**: Each microservice owns its database
- **API Gateway Ready**: Services expose REST APIs via minimal endpoints
- **Inter-Service Communication**: gRPC for internal service calls
- **Caching Strategy**: Redis for performance optimization
- **Health Monitoring**: Built-in health checks for all services

## 🚀 Services

### 1. Catalog Service
**Responsibility**: Product catalog management

- **Database**: PostgreSQL with Marten (Event Sourcing)
- **Operations**: CRUD operations on products
- **Domain Model**: Product (Id, Name, Category, Description, ImageFile, Price)

**Key Features**:
- Get all products with pagination
- Get product by ID
- Get products by category
- Create, update, and delete products

---

### 2. Basket Service
**Responsibility**: Shopping cart management

- **Database**: PostgreSQL with Marten
- **Cache**: Redis with Decorator Pattern
- **Integration**: Discount Service (gRPC)

**Key Features**:
- Get shopping basket by username
- Store/update basket items
- Delete basket
- Automatic discount application via gRPC call
- Cached repository for improved performance

**Architecture Highlight**:
```csharp
CachedBasketRepository (Decorator)
    ↓
BasketRepository (Base)
    ↓
PostgreSQL + Redis
```

---

### 3. Discount Service
**Responsibility**: Discount and coupon management

- **Database**: SQLite
- **Protocol**: gRPC (binary, efficient)
- **Model**: CouponModel (Id, ProductName, Description, Amount)

**Key Features**:
- Get discount by product name
- Create new discounts
- Update existing discounts
- Delete discounts
- High-performance gRPC endpoints

---

### 4. Ordering Service
**Responsibility**: Order processing and management

- **Database**: SQL Server with Entity Framework Core
- **Architecture**: Clean Architecture (4-layer DDD)

**Layers**:
1. **Domain Layer**: Rich domain models, value objects, aggregates
2. **Application Layer**: CQRS commands/queries, DTOs, handlers
3. **Infrastructure Layer**: Data access, EF configurations, interceptors
4. **API Layer**: HTTP endpoints, dependency injection

**Domain Models**:
- `Order` (Aggregate Root)
- `Customer`
- `OrderItem`
- Value Objects: `OrderId`, `CustomerId`, `OrderName`, `Address`, `Payment`
- Enums: `OrderStatus` (Pending, Completed, etc.)

**Operations**:
- **Commands**: CreateOrder, UpdateOrder, DeleteOrder
- **Queries**: GetOrders, GetOrdersByCustomer, GetOrdersByName

## 🛠️ Technologies

### Core Framework
- **.NET 9.0** - Latest .NET runtime with performance improvements
- **ASP.NET Core** - Web framework for building APIs

### Key Libraries
| Library | Version | Purpose |
|---------|---------|---------|
| **MediatR** | 13.0.0 | CQRS and Mediator pattern |
| **Carter** | 9.0.0 | Minimal API routing |
| **Marten** | 8.6.0 | Event sourcing & document DB |
| **Entity Framework Core** | 9.0.10 | ORM for relational databases |
| **FluentValidation** | 12.0.0 | Request validation |
| **Mapster** | 7.4.0 | Object-to-object mapping |
| **gRPC** | 2.71.0 | Inter-service communication |
| **StackExchange.Redis** | - | Distributed caching |

### Data Stores
- **PostgreSQL** - Catalog & Basket services
- **SQL Server** - Ordering service
- **SQLite** - Discount service
- **Redis** - Distributed cache

## 🎨 Key Patterns

### CQRS (Command Query Responsibility Segregation)
Separates read and write operations for better scalability and maintainability.

```csharp
// Command
public record CreateOrderCommand(OrderDto Order) : ICommand<CreateOrderResult>;

// Query
public record GetOrdersQuery(PaginationRequest PaginationRequest)
    : IQuery<GetOrdersResult>;
```

### Domain-Driven Design (DDD)
Rich domain models with business logic encapsulation.

```
Ordering Service:
├── Domain/          # Entities, Value Objects, Aggregates
├── Application/     # Use Cases, DTOs, Handlers
├── Infrastructure/  # Data Access, External Services
└── API/            # HTTP Endpoints, Controllers
```

### Repository Pattern with Decorator
Abstracted data access with transparent caching layer.

```csharp
public class CachedBasketRepository : IBasketRepository
{
    private readonly BasketRepository _repository;
    private readonly IDistributedCache _cache;

    // Decorator adds caching behavior
}
```

### Minimal APIs with Carter
Lightweight, modular endpoint definition.

```csharp
public class CreateProductEndpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost("/products", async (CreateProductCommand command) =>
        {
            // Handler logic
        });
    }
}
```

## 📁 Project Structure

```
EShop/
├── src/
│   ├── BuildingBlocks/
│   │   └── BuildingBlocks/              # Shared library
│   │       ├── CQRS/                    # Command/Query interfaces
│   │       ├── Behaviors/               # MediatR behaviors
│   │       ├── Exceptions/              # Custom exceptions
│   │       └── Pagination/              # Pagination support
│   │
│   ├── Services/
│   │   ├── Catalog/
│   │   │   └── Catalog.API/             # Catalog microservice
│   │   │
│   │   ├── Basket/
│   │   │   └── Basket.API/              # Basket microservice
│   │   │
│   │   ├── Discount/
│   │   │   └── Discount.Grpc/           # Discount gRPC service
│   │   │
│   │   └── Ordering/
│   │       ├── Ordering.API/            # API layer
│   │       ├── Ordering.Application/    # Application layer
│   │       ├── Ordering.Domain/         # Domain layer
│   │       └── Ordering.Infrastructure/ # Infrastructure layer
│   │
│   └── eshop-microservices.sln         # Solution file
│
└── README.md
```

## 🚦 Getting Started

### Prerequisites
- .NET 9.0 SDK
- Docker Desktop
- PostgreSQL
- SQL Server
- Redis

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/eshop-microservices.git
   cd eshop-microservices
   ```

2. **Restore dependencies**
   ```bash
   dotnet restore src/eshop-microservices.sln
   ```

3. **Start infrastructure services** (PostgreSQL, Redis, SQL Server)
   ```bash
   docker-compose up -d
   ```

4. **Run migrations**
   ```bash
   # Ordering Service
   cd src/Services/Ordering/Ordering.API
   dotnet ef database update
   ```

5. **Run services**
   ```bash
   # Terminal 1 - Catalog Service
   cd src/Services/Catalog/Catalog.API
   dotnet run

   # Terminal 2 - Basket Service
   cd src/Services/Basket/Basket.API
   dotnet run

   # Terminal 3 - Discount Service
   cd src/Services/Discount/Discount.Grpc
   dotnet run

   # Terminal 4 - Ordering Service
   cd src/Services/Ordering/Ordering.API
   dotnet run
   ```

6. **Access services**
   - Catalog API: `https://localhost:5001`
   - Basket API: `https://localhost:5002`
   - Discount gRPC: `https://localhost:5003`
   - Ordering API: `https://localhost:5004`
   - Health Checks: `https://localhost:500X/health`

## 💾 Database Strategy

Each service uses the most appropriate database technology:

| Service | Database | Reason |
|---------|----------|--------|
| **Catalog** | PostgreSQL + Marten | Event sourcing, flexible schema |
| **Basket** | PostgreSQL + Marten + Redis | Event sourcing + caching for performance |
| **Discount** | SQLite | Lightweight, simple data structure |
| **Ordering** | SQL Server + EF Core | Complex relationships, ACID transactions |

## 📊 Health Checks

All services include health check endpoints for monitoring:

```
GET /health
```

Monitors:
- Database connectivity (PostgreSQL, SQL Server, SQLite)
- Redis cache connectivity
- Service dependencies

## 🔄 Inter-Service Communication

**gRPC** is used for synchronous service-to-service communication:

- **Basket → Discount**: Applies discounts when retrieving basket
- High performance binary protocol
- Strongly-typed contracts via Protocol Buffers

## 🏆 Best Practices Demonstrated

- ✅ Separation of Concerns
- ✅ Single Responsibility Principle
- ✅ Dependency Injection
- ✅ Async/Await patterns
- ✅ Request validation with FluentValidation
- ✅ Global exception handling
- ✅ Logging with structured logs
- ✅ Health monitoring
- ✅ Containerization ready
- ✅ Clean Architecture
- ✅ Domain-Driven Design

## 📝 License

This project is licensed under the MIT License.

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

---

**Built with ❤️ using .NET 9.0**