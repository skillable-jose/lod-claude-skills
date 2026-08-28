# Common Patterns

## SOLID Principles

### Single Responsibility Principle (SRP)
A class should have one, and only one, reason to change.

**Anti-pattern**: God class doing everything
```
class UserManager {
  validateUser()
  saveToDatabase()
  sendEmail()
  generateReport()
  logActivity()
}
```

**Correct**: Separate concerns
```
class UserValidator { validateUser() }
class UserRepository { save() }
class EmailService { send() }
class ReportGenerator { generate() }
class ActivityLogger { log() }
```

### Open/Closed Principle (OCP)
Software entities should be open for extension but closed for modification.

Use interfaces/abstractions to allow extension without modifying existing code.

### Liskov Substitution Principle (LSP)
Subtypes must be substitutable for their base types without altering correctness.

### Interface Segregation Principle (ISP)
No client should be forced to depend on methods it does not use. Create focused, specific interfaces.

### Dependency Inversion Principle (DIP)
Depend on abstractions, not concretions. High-level modules should not depend on low-level modules.

## Clean Architecture Layers

Organize code into clean, dependency-flowing layers:

```
┌─────────────────────────────────────┐
│         Presentation Layer          │  ← Controllers, Views, API endpoints
│         (UI, API, CLI)              │
└──────────────┬──────────────────────┘
               │ depends on ↓
┌──────────────┴──────────────────────┐
│       Application Layer             │  ← Use cases, business logic
│    (Services, Commands, Queries)    │
└──────────────┬──────────────────────┘
               │ depends on ↓
┌──────────────┴──────────────────────┐
│         Domain Layer                │  ← Entities, value objects, domain logic
│    (Business rules, no dependencies)│
└──────────────△──────────────────────┘
               │ implemented by
┌──────────────┴──────────────────────┐
│      Infrastructure Layer           │  ← Database, external services, file I/O
│   (EF Core, HTTP clients, storage)  │
└─────────────────────────────────────┘
```

**Dependency Rule**: Inner layers NEVER depend on outer layers.

## Skeleton Projects

When implementing new functionality:
1. Search for battle-tested skeleton projects
2. Use parallel agents to evaluate options:
   - Security assessment
   - Extensibility analysis
   - Relevance scoring
   - Implementation planning
3. Clone best match as foundation
4. Iterate within proven structure

## Anemic Domain Model

Domain classes are plain data containers (properties only). All business logic lives in the **Application/Service layer**.

- ✅ Domain model holds data — services hold behavior
- ❌ Do NOT put business methods on domain entities (no rich domain model)

## Design Patterns

### Repository Pattern

Encapsulate data access behind a consistent interface:

**Naming Convention** (Repositories only — does NOT apply to Services):
Method names MUST start with entity type to enable grouping by entity:
- `{Entity}FindAll()` or `{Entity}GetAllAsync()` - Retrieve all entities
- `{Entity}SingleById(id)` or `{Entity}SingleByIdAsync(id)` - Retrieve single entity; **throws NotFoundException** if not found
- `{Entity}SingleOrDefaultById(id)` or `{Entity}SingleOrDefaultByIdAsync(id)` - Retrieve single entity; **returns null** if not found
- `{Entity}Create(data)` or `{Entity}AddAsync(data)` - Create new entity
- `{Entity}Update(id, data)` or `{Entity}UpdateAsync(id, data)` - Update existing entity
- `{Entity}Delete(id)` or `{Entity}DeleteAsync(id)` - Delete entity

**Single vs SingleOrDefault (CRITICAL)**:
- **Single** methods throw `NotFoundException` when the entity does not exist — use when the caller expects the entity to be present (e.g., update, delete, or downstream logic that requires it)
- **SingleOrDefault** methods return a falsey value (null) when the entity does not exist — use when absence is a valid outcome (e.g., lookup before create, conditional logic)

**Examples**:
- `UserFindAll()`, `UserSingleById(123)`, `UserCreate(userData)`
- `OrderFindAll()`, `OrderSingleOrDefaultById(456)`, `OrderUpdate(456, orderData)`

**Domain vs Entity Separation (CRITICAL)**:
- Repository interfaces operate on **domain model classes** (defined in Abstractions), NOT on ORM entity classes
- ORM entity classes (navigation properties, `[Table]` attributes, etc.) are an implementation detail of the Repository layer and MUST NOT leak outside it
- Repository implementations map between domain models and ORM entities internally
- Services and other consumers never see or depend on ORM entity classes

**Repository Contract**:
- Read methods return **new domain model instances** mapped from persistence — callers never receive tracked ORM objects
- Write methods accept domain models, map them to ORM entities internally, perform persistence, and return **new domain model instances** reflecting the saved state
- The ORM's change-tracking, identity map, and mutation are confined to the repository implementation — they are invisible to callers
- ✅ `user = UserSingleById(id)` — Throws NotFoundException if not found
- ✅ `user = UserSingleOrDefaultById(id)` — Returns null if not found
- ✅ `newUser = UserCreate(userData)` — Returns a new domain model mapped from the saved entity
- ✅ `updatedUser = UserUpdate(id, changes)` — Returns a new domain model reflecting persisted changes
- ❌ Returning tracked ORM entities to callers — FORBIDDEN
- ❌ Accepting or returning ORM entity types in the repository interface — FORBIDDEN

**Benefits**:
- Methods grouped by entity type in IDE autocomplete
- Business logic doesn't know about storage mechanism or ORM behavior
- Easy to swap data sources (SQL → NoSQL → API) — callers are persistence-ignorant
- Simplified testing with mocks — no change-tracker concerns in tests
- Predictable, testable code — callers work with plain domain objects

See language-specific rules (csharp, typescript) for implementation details.

### API Response Format

Use a consistent envelope for all API responses:

```
{
  "success": boolean,    // true for successful operations, false for errors
  "data": T | null,      // Response payload (null on error)
  "error": string | null,// Error message (null on success)
  "statusCode": number   // REQUIRED — HTTP status code (e.g., 200, 404, 500). Defaults to 200 for success responses.
}
```

**CRITICAL**: `statusCode` is ALWAYS required on every response. For success responses, use HTTP 200 (OK) unless a different status applies. For failure responses, always specify the appropriate HTTP error code.

**Benefits**:
- Consistent client-side error handling
- Clear success/failure indication
- Programmatic status handling via `statusCode`