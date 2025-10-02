# Template: Crear Nueva Feature

## Contexto
Este proyecto usa **Vertical Slice Architecture** con **CQRS**. Cada feature debe ser autocontenida.

## Información Requerida

Antes de crear una feature, especifica:

1. **Microservicio**: ¿Catalog.API, Basket.API, u otro?
2. **Tipo**: ¿Command (escritura) o Query (lectura)?
3. **Nombre**: ¿Qué hace? (ej: GetProductsByPrice, UpdateOrderStatus)
4. **Parámetros**: ¿Qué datos recibe?
5. **Respuesta**: ¿Qué devuelve?

## Estructura a Crear

Para cada feature, crear:

```
[ServiceName].API/
└── [EntityPlural]/
    └── [FeatureName]/
        ├── [FeatureName]Endpoint.cs      # Carter module con MapPost/Get/Put/Delete
        ├── [FeatureName]Handler.cs       # MediatR handler con lógica
        └── [FeatureName]Validator.cs     # FluentValidation (si aplica)
```

## Checklist de Implementación

### 1. Command/Query Record
```csharp
// Para Command
public record [FeatureName]Command([params])
    : ICommand<[FeatureName]Result>;

// Para Query
public record [FeatureName]Query([params])
    : IQuery<[FeatureName]Result>;
```

### 2. Result Record
```csharp
public record [FeatureName]Result([properties]);
```

### 3. Validator (opcional pero recomendado)
```csharp
public class [FeatureName]Validator
    : AbstractValidator<[FeatureName]Command>
{
    public [FeatureName]Validator()
    {
        RuleFor(x => x.Property)
            .NotEmpty()
            .WithMessage("Message");
    }
}
```

### 4. Handler
```csharp
public class [FeatureName]Handler(IDocumentSession session)
    : ICommandHandler<[FeatureName]Command, [FeatureName]Result>
{
    public async Task<[FeatureName]Result> Handle(
        [FeatureName]Command command,
        CancellationToken cancellationToken)
    {
        // 1. Lógica de negocio
        // 2. Persistencia con session
        // 3. Return result
    }
}
```

### 5. Endpoint (Carter)
```csharp
public class [FeatureName]Endpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.Map[Verb]("/[route]", async ([params], ISender sender) =>
        {
            var command = new [FeatureName]Command([params]);
            var result = await sender.Send(command);
            return Results.[StatusCode](result);
        })
        .WithName("[FeatureName]")
        .Produces<[FeatureName]Result>(StatusCodes.Status[Code])
        .ProducesProblem(StatusCodes.Status400BadRequest)
        .WithSummary("[Summary]")
        .WithDescription("[Description]");
    }
}
```

## Ejemplos de Features Existentes

### Command (Escritura)
Ver: `src/Services/Catalog/Catalog.API/Products/CreateProduct/`

### Query Simple
Ver: `src/Services/Catalog/Catalog.API/Products/GetProductById/`

### Query con Filtros
Ver: `src/Services/Catalog/Catalog.API/Products/GetProductByCategory/`

### Query con Paginación
Ver: `src/Services/Catalog/Catalog.API/Products/GetProducts/`

## Validaciones Comunes

```csharp
// NotEmpty
RuleFor(x => x.Name).NotEmpty();

// Longitud
RuleFor(x => x.Name).Length(3, 100);

// Rango numérico
RuleFor(x => x.Price).GreaterThan(0);

// Email
RuleFor(x => x.Email).EmailAddress();

// Custom
RuleFor(x => x.Category)
    .Must(BeValidCategory)
    .WithMessage("Invalid category");
```

## Manejo de Errores

Lanza excepciones personalizadas según el caso:

```csharp
// No encontrado
throw new NotFoundException("Product", productId);

// Validación de negocio
throw new BadRequestException("Stock insufficient");

// Error interno
throw new InternalServerException("Database error", ex.Message);
```

## Testing (Futuro)

Estructura para tests:
```
[FeatureName]Tests.cs
├── Handle_ValidRequest_ReturnsSuccess()
├── Handle_InvalidRequest_ThrowsValidationException()
└── Handle_NotFound_ThrowsNotFoundException()
```

## Prompt de Ejemplo

```
Crea una nueva feature "GetProductsByPriceRange" en Catalog.API que:
- Sea una Query
- Reciba: decimal minPrice, decimal maxPrice
- Devuelva: lista de productos en ese rango
- Incluya validación: minPrice > 0, maxPrice > minPrice
- Use paginación (pageNumber, pageSize)
- Siga el patrón de features existentes
```
