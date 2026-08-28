---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# TypeScript/JavaScript Coding Style

> This file extends [common/coding-style.md](../common/coding-style.md) with TypeScript/JavaScript specific content.

## Immutability

Use spread operator for immutable updates:

```typescript
// WRONG: Mutation
function updateUser(user, name) {
  user.name = name  // MUTATION!
  return user
}

// CORRECT: Immutability
function updateUser(user, name) {
  return {
    ...user,
    name
  }
}
```

## Error Handling

Let exceptions bubble to the global handler (see [common/logging.md](../common/logging.md#no-trycatch-log-pattern)). Only catch locally when **transforming** to a domain-specific result:

```typescript
// WRONG — catch-log-rethrow; telemetry already captures this
try {
  return await riskyOperation()
} catch (error) {
  console.error('Operation failed:', error)  // ❌ console.error in production
  throw new Error('Detailed message')        // ❌ redundant rethrow
}

// CORRECT — no try/catch; let global handler capture exceptions
const result = await riskyOperation()
return result

// CORRECT — catch only to transform into a domain result
try {
  return Result.success(await riskyOperation())
} catch (error) {
  if (error instanceof NotFoundException) {
    return Result.failure('Resource not found')
  }
  throw error  // re-throw unexpected errors to global handler
}
```

## Input Validation

Use Zod for schema-based validation:

```typescript
import { z } from 'zod'

const schema = z.object({
  email: z.string().email(),
  age: z.number().int().min(0).max(150)
})

const validated = schema.parse(input)
```

## Console.log

- No `console.log` statements in production code
- Use proper logging libraries instead
- See hooks for automatic detection
