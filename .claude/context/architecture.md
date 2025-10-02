# Arquitectura del Proyecto EShop Microservices

## Visión General

Este proyecto sigue una arquitectura de **microservicios** con **CQRS** y **Vertical Slice Architecture**.

## Principios Arquitectónicos

### 1. Vertical Slice Architecture
- Cada feature es una "rebanada vertical" completa
- Incluye: Request → Handler → Validación → Respuesta
- Organizada en carpetas por feature (no por capas técnicas)

```
Products/
├── CreateProduct/
│   ├── CreateProductEndpoint.cs      # Carter endpoint
│   ├── CreateProductHandler.cs       # MediatR handler
│   └── CreateProductValidator.cs     # FluentValidation
├── GetProductById/
└── UpdateProduct/
```

### 2. CQRS (Command Query Responsibility Segregation)

**Commands** (Escritura):
- Modifican estado
- No devuelven datos (solo Result)
- Ejemplos: CreateProduct, UpdateProduct, DeleteProduct

**Queries** (Lectura):
- Solo leen datos
- No modifican estado
- Ejemplos: GetProducts, GetProductById

### 3. Independencia de Microservicios
- Cada servicio tiene su **propia base de datos**
- No hay llamadas directas entre servicios (futuro: message bus)
- Comunicación eventual mediante eventos

## Stack Tecnológico

### APIs
- **ASP.NET Core** 8.0 (Minimal APIs)
- **Carter** - Organización de endpoints
- **MediatR** - Patrón Mediator para CQRS

### Persistencia
- **Marten** - Document DB sobre PostgreSQL
- **PostgreSQL** - Base de datos relacional/documental

### Validación & Errores
- **FluentValidation** - Validaciones declarativas
- **CustomExceptionHandler** - Manejo global de errores

### Infraestructura
- **Docker Compose** - Orquestación de contenedores
- **Health Checks** - Monitoreo de salud de servicios

## BuildingBlocks (Librería Compartida)

Código reutilizable entre microservicios:

```
BuildingBlocks/
├── Behaviors/
│   ├── ValidationBehavior.cs    # Pipeline de validación MediatR
│   └── LoggingBehavior.cs       # Pipeline de logging
├── CQRS/
│   ├── ICommand.cs              # Interfaz para commands
│   └── IQuery.cs                # Interfaz para queries
└── Exceptions/
    ├── Handler/
    │   └── CustomExceptionHandler.cs
    ├── NotFoundException.cs
    ├── BadRequestException.cs
    └── InternalServerException.cs
```

## Flujo de una Request

1. **Cliente** → HTTP Request
2. **Carter Endpoint** → Recibe y mapea request
3. **MediatR** → Envía command/query
4. **ValidationBehavior** → Valida con FluentValidation
5. **LoggingBehavior** → Log antes/después
6. **Handler** → Lógica de negocio
7. **Marten/PostgreSQL** → Persistencia
8. **Response** → Devuelve resultado

## Configuración de Servicios (Program.cs)

Todos los servicios siguen este patrón:

```csharp
// 1. Carter para endpoints
builder.Services.AddCarter();

// 2. MediatR con behaviors
builder.Services.AddMediatR(config => {
    config.RegisterServicesFromAssembly(assembly);
    config.AddOpenBehavior(typeof(ValidationBehavior<,>));
    config.AddOpenBehavior(typeof(LoggingBehavior<,>));
});

// 3. Validadores
builder.Services.AddValidatorsFromAssembly(assembly);

// 4. Marten (Document DB)
builder.Services.AddMarten(opts => {
    opts.Connection(connectionString);
}).UseLightweightSessions();

// 5. Exception Handler
builder.Services.AddExceptionHandler<CustomExceptionHandler>();

// 6. Health Checks
builder.Services.AddHealthChecks()
    .AddNpgSql(connectionString);
```

## Convenciones de Código

### Naming
- **Commands**: `[Verb][Entity]Command` (ej: `CreateProductCommand`)
- **Queries**: `Get[Entity(s)][Filter]Query` (ej: `GetProductByIdQuery`)
- **Handlers**: `[CommandOrQuery]Handler`
- **Endpoints**: `[Feature]Endpoint`

### Estructura de Feature
```csharp
// 1. Record para Command/Query
public record CreateProductCommand(...) : ICommand<CreateProductResult>;

// 2. Record para Result
public record CreateProductResult(Guid Id);

// 3. Validator
public class CreateProductValidator : AbstractValidator<CreateProductCommand> { }

// 4. Handler
public class CreateProductHandler : ICommandHandler<CreateProductCommand, CreateProductResult> { }

// 5. Endpoint (Carter)
public class CreateProductEndpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost("/products", async (CreateProductCommand command, ISender sender) =>
        {
            var result = await sender.Send(command);
            return Results.Created($"/products/{result.Id}", result);
        });
    }
}
```

## Base de Datos

### Marten como Document DB
- Almacena objetos como JSON en PostgreSQL
- Sin migraciones complejas
- Schema automático basado en clases C#

### Configuración típica
```csharp
builder.Services.AddMarten(opts =>
{
    opts.Connection(connectionString);

    // Configurar identidad
    opts.Schema.For<Product>().Identity(x => x.Id);

}).UseLightweightSessions();
```

## Próximos Pasos Arquitectónicos

1. **Message Bus** (RabbitMQ/MassTransit) para comunicación entre servicios
2. **API Gateway** (YARP) como punto de entrada único
3. **Service Discovery** para ambientes dinámicos
4. **Distributed Tracing** (OpenTelemetry)
5. **Event Sourcing** para algunos agregados críticos
