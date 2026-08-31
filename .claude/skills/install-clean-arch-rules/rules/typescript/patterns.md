---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# TypeScript/JavaScript Patterns

> This file extends [common/patterns.md](../common/patterns.md) with TypeScript/JavaScript specific content.

## API Response Format

> Normative per the org's frontend-architecture decision (see [frontend-arch.md](frontend-arch.md)) — a discriminated union, not a single shape with optional fields. Prefer `import type { ApiResponse } from '@skillr/api-types'` (or the project's equivalent shared package) over redeclaring this locally.

```typescript
type ApiResponse<T> =
  | { success: true; data: T }
  | { success: false; error: { code: string; message: string; details?: unknown } }

type PaginatedApiResponse<T> =
  | { success: true; data: T[]; pagination: { page: number; pageSize: number; totalCount: number; hasMore: boolean } }
  | { success: false; error: { code: string; message: string; details?: unknown } }
```

Narrow on `response.success` — TypeScript then knows `data` exists in the `true` branch and `error` exists in the `false` branch. Don't add a `statusCode` field to this type: HTTP status is a transport-layer detail the shared `shared/api/client.ts` already consumed to decide `success`; a caller matches on `error.code`, never on a status number or on `error.message` text.

## Custom Hooks Pattern

```typescript
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}
```

## Repository Pattern (non-baseline — see note)

> **The org's actual React baseline is plain functions in a feature's `api/` module** (see [frontend-arch.md](frontend-arch.md) and [react.md](react.md)), not a class-based repository with a DI'd interface. The pattern below is a legitimate alternative for a project that deliberately chooses class-based DI (e.g. a larger Node backend-for-frontend, or a team porting backend-style layering on purpose) — it is not what to reach for by default in a React feature folder. Record the choice in that project's `ADR-0000` if you do use it.

**Naming Convention** (Repositories only — does NOT apply to Services): Methods MUST start with entity type for grouping (e.g., `userSingleById`, `userCreate`). Service interfaces use natural application-level naming (e.g., `getUserById`, `createUser`).

**Single vs SingleOrDefault**: `Single` methods throw `NotFoundException` when the entity is not found. `SingleOrDefault` methods return `null`. Use `Single` when absence is exceptional; use `SingleOrDefault` when absence is a valid outcome.

**Immutability**: Repository methods MUST NOT modify passed-in objects - always return new instances.

```typescript
// Standalone entity-specific repository (no generic Repository<T> base —
// entity-first naming makes a shared base impractical and causes DRY collisions)
// Method names start with entity type for IDE autocomplete grouping
// Single throws NotFoundException; SingleOrDefault returns null
interface UserRepository {
  userSingleById(id: string): Promise<User>
  userSingleOrDefaultById(id: string): Promise<User | null>
  userSingleByEmail(email: string): Promise<User>
  userSingleOrDefaultByEmail(email: string): Promise<User | null>
  userFindAll(filters?: UserFilters): Promise<readonly User[]>
  userFindActive(): Promise<readonly User[]>
  userCreate(data: CreateUserDto): Promise<User>
  userUpdate(id: string, data: UpdateUserDto): Promise<User>
  userDelete(id: string): Promise<void>
}

// Implementation example with Prisma
class PrismaUserRepository implements UserRepository {
  constructor(private prisma: PrismaClient) {}

  // Single — throws NotFoundException when not found
  async userSingleById(id: string): Promise<User> {
    const user = await this.userSingleOrDefaultById(id)
    if (!user) throw new NotFoundException(`User not found (UserId: ${id}).`)
    return user
  }

  async userSingleByEmail(email: string): Promise<User> {
    const user = await this.userSingleOrDefaultByEmail(email)
    if (!user) throw new NotFoundException(`User not found (Email: ${email}).`)
    return user
  }

  // SingleOrDefault — returns null when not found
  async userSingleOrDefaultById(id: string): Promise<User | null> {
    return await this.prisma.user.findUnique({ where: { id } })
  }

  async userSingleOrDefaultByEmail(email: string): Promise<User | null> {
    return await this.prisma.user.findUnique({ where: { email } })
  }

  async userFindAll(filters?: UserFilters): Promise<readonly User[]> {
    return await this.prisma.user.findMany({ where: filters })
  }

  async userFindActive(): Promise<readonly User[]> {
    return await this.prisma.user.findMany({ where: { isActive: true } })
  }

  async userCreate(data: CreateUserDto): Promise<User> {
    return await this.prisma.user.create({ data })
  }

  async userUpdate(id: string, data: UpdateUserDto): Promise<User> {
    return await this.prisma.user.update({ where: { id }, data })
  }

  async userDelete(id: string): Promise<void> {
    await this.prisma.user.delete({ where: { id } })
  }
}
```

**Benefits of entity-first naming**:
- Methods grouped by entity in IDE autocomplete
- Easy to discover all User-related methods: `user...` shows all options (e.g., `userSingle...`, `userCreate`, `userUpdate`)
- Clear separation between different entity operations
