# Agentes Claude para EShop Microservices

Este archivo documenta los agentes especializados disponibles para trabajar con este proyecto de microservicios.

## Agentes Disponibles

### 1. Microservice Feature Builder
**Propósito:** Crear nuevas features siguiendo el patrón Vertical Slice Architecture

**Cuándo usar:**
- Agregar nuevos endpoints a Catalog o Basket
- Crear operaciones CRUD completas
- Implementar queries o commands con MediatR

**Ejemplo de prompt:**
```
Crea una nueva feature "GetProductsByPrice" en Catalog.API que filtre productos por rango de precio usando el patrón establecido
```

### 2. Exception Handler Specialist
**Propósito:** Extender o modificar el manejo de excepciones centralizado

**Cuándo usar:**
- Agregar nuevos tipos de excepciones personalizadas
- Modificar respuestas de error
- Implementar logging específico

**Ejemplo de prompt:**
```
Agrega una excepción ConflictException que retorne 409 cuando haya duplicados
```

### 3. Database Migration Agent
**Propósito:** Manejar cambios en modelos y datos semilla (seed data)

**Cuándo usar:**
- Modificar modelos de dominio (Product, ShoppingCart)
- Actualizar datos iniciales (CatalogInitialData)
- Configurar Marten Document DB

**Ejemplo de prompt:**
```
Modifica el modelo Product para incluir una propiedad "Stock" y actualiza los datos iniciales
```

### 4. Docker & Infrastructure Agent
**Propósito:** Configurar contenedores y servicios de infraestructura

**Cuándo usar:**
- Agregar nuevos microservicios al docker-compose
- Configurar nuevas bases de datos
- Modificar health checks

**Ejemplo de prompt:**
```
Agrega un nuevo microservicio "Ordering.API" con su propia base de datos PostgreSQL
```

### 5. Validation & Behaviors Agent
**Propósito:** Implementar validaciones y behaviors de MediatR

**Cuándo usar:**
- Crear validadores con FluentValidation
- Agregar nuevos behaviors (Logging, Validation, etc.)
- Implementar reglas de negocio complejas

**Ejemplo de prompt:**
```
Crea validadores para asegurar que el precio de productos sea mayor a 0 y el nombre tenga entre 3-100 caracteres
```

## Estructura de Carpetas para Agentes

```
.claude/
├── agents.md                    # Este archivo
├── prompts/
│   ├── new-feature.md          # Template para nuevas features
│   ├── new-microservice.md     # Template para nuevos servicios
│   └── exception-handling.md   # Template para manejo de errores
└── context/
    ├── architecture.md         # Explicación de la arquitectura
    └── patterns.md             # Patrones usados (CQRS, Vertical Slice)
```

## Mejores Prácticas

1. **Siempre especifica el microservicio** donde trabajarás (Catalog.API, Basket.API)
2. **Menciona los patrones** que debes seguir (Vertical Slice, CQRS)
3. **Incluye contexto** de features similares existentes
4. **Solicita tests** si es necesario
5. **Pide validación** del cumplimiento de patrones

## Comandos Útiles para Agentes

```bash
# Ver estructura de un microservicio
ls -la src/Services/[ServiceName]/[ServiceName].API/

# Ver features existentes
ls -la src/Services/Catalog/Catalog.API/Products/

# Verificar BuildingBlocks compartidos
ls -la src/BuildingBlocks/BuildingBlocks/
```

## Ejemplos de Tareas Complejas

### Crear un nuevo microservicio completo
```
Usando el agente "Docker & Infrastructure Agent" y "Microservice Feature Builder",
crea un nuevo microservicio "Ordering.API" con:
- Base de datos PostgreSQL
- Pattern CQRS con MediatR
- Features: CreateOrder, GetOrderById, GetOrdersByUser
- Integrado en docker-compose
```

### Refactorizar manejo de errores
```
Usando el agente "Exception Handler Specialist",
refactoriza el CustomExceptionHandler para incluir:
- ConflictException (409)
- UnauthorizedException (401)
- Logging estructurado con información adicional
```
