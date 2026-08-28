---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.spec.ts"
  - "**/*.test.ts"
  - "**/*.spec.tsx"
  - "**/*.test.tsx"
---
# TypeScript/JavaScript Testing

> This file extends [common/testing.md](../common/testing.md) with TypeScript/JavaScript-specific conventions. Read that file first for the intent rule, TDD workflow, AAA pattern, and coverage requirements.

---

## Test Framework & Tools

| Concern | React | Next.js | Angular |
|---|---|---|---|
| Unit/integration runner | **Vitest** | **Vitest** | **Jest** (default) or Vitest |
| Component rendering | **React Testing Library** | **React Testing Library** (Server Components: call the async function directly, no render needed) | **Angular Testing Library** / `TestBed` |
| Mocking | `vi.fn()` / `vi.spyOn()` | `vi.fn()` / `vi.spyOn()` | `jest.fn()` / `jest.spyOn()` |
| E2E | **Playwright** | **Playwright** | **Playwright** |
| Assertions | Vitest `expect` + Testing Library matchers | Vitest `expect` + Testing Library matchers | Jest `expect` + Testing Library matchers |

---

## Test Naming Convention

TypeScript tests do not use method names — they use nested `describe`/`it` string blocks. The three-part intent rule from [common/testing.md](../common/testing.md) maps to the nesting structure:

| Intent part | Maps to |
|---|---|
| What is under test | Outer `describe` — class or component name |
| Method or behaviour under test | Inner `describe` — method name or user action |
| Outcome + condition | `it('should ... when ...')` string |

### Service / Class Tests

```typescript
// user-service.test.ts
describe('UserService', () => {
  describe('getUserById', () => {
    it('should return the user when the user exists', async () => { ... })
    it('should return null when the user does not exist', async () => { ... })
    it('should throw UnauthorizedError when the caller is not authenticated', async () => { ... })
  })

  describe('createUser', () => {
    it('should return the created user when the request is valid', async () => { ... })
    it('should return a failure result when the email is already in use', async () => { ... })
    it('should return a failure result when the email format is invalid', async () => { ... })
  })

  describe('deleteUser', () => {
    it('should delete the user when the user exists and is not an admin', async () => { ... })
    it('should throw ValidationError when the user is an admin', async () => { ... })
  })
})
```

### React Component Tests — User Perspective

React component tests express behaviour from the **user's perspective**, not the implementation's. The inner `describe` names a user action or rendered state, not a method name.

```typescript
// user-card.test.tsx
describe('UserCard', () => {
  it('renders the user name and email', () => { ... })
  it('renders the featured badge when featured is true', () => { ... })
  it('does not render the featured badge when featured is false', () => { ... })

  describe('when the delete button is clicked', () => {
    it('calls onDelete with the user id', () => { ... })
    it('disables the delete button while deletion is pending', () => { ... })
  })
})
```

### Angular Component Tests

Angular component tests follow the same describe/it structure. Use `TestBed` for component tests and plain class instantiation for service tests.

```typescript
// user-list-view.component.spec.ts
describe('UserListViewComponent', () => {
  it('renders a list item for each user', () => { ... })
  it('renders the empty state when users is an empty array', () => { ... })
  it('renders the loading spinner when isLoading is true', () => { ... })

  describe('when a user row delete button is clicked', () => {
    it('emits deleteRequest with the user id', () => { ... })
  })
})
```

**Rules:**
- Outer `describe` = class or component name — always matches the file's subject exactly
- Inner `describe` = method name (services) or user action / rendered state (components)
- `it` string starts with `'should'` and reads as a complete sentence
- Vague names are forbidden: `it('works')`, `it('handles error')`, `it('test 1')` — all forbidden
- The full intent must be readable from `describe` + `it` without opening the test body

---

## File Co-location

Tests live alongside the source file they test — not in a separate `__tests__` folder.

```
components/users/
  user-card/
    user-card.tsx
    user-card.test.tsx            ← React component test
    user-card.module.css
pages/users/
  users-page.tsx
  users-page.test.tsx
state/users/
  use-user-list.ts
  use-user-list.test.ts
services/users/
  user-service.ts
  user-service.test.ts
```

Angular uses `.spec.ts` by convention:

```
components/users/
  user-card/
    user-card.component.ts
    user-card.component.spec.ts    ← Angular component test
    user-card.component.html
    user-card.component.scss
services/users/
  user.service.ts
  user.service.spec.ts
```

---

## AAA Pattern

Every test follows Arrange / Act / Assert with labelled comments:

```typescript
it('should return the user when the user exists', async () => {
  // Arrange
  const userId = 'user-1'
  const expectedUser: User = { id: userId, name: 'Alice', email: 'alice@example.com', role: 'member', isActive: true, createdAt: new Date() }
  mockUserRepository.userSingleOrDefaultById.mockResolvedValue(expectedUser)

  // Act
  const result = await userService.getUserById(userId)

  // Assert
  expect(result).toEqual(expectedUser)
  expect(mockUserRepository.userSingleOrDefaultById).toHaveBeenCalledOnce()
  expect(mockUserRepository.userSingleOrDefaultById).toHaveBeenCalledWith(userId)
})
```

---

## Mocking

### Services — Mock the Repository Interface

Services are plain classes. Construct them directly with a mock repository. Do not use `vi.mock` module-level patching — construct the mock object explicitly.

```typescript
// user-service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { UserService } from './user-service'
import type { IUserRepository } from '../domain/interfaces/i-user-repository'

const mockUserRepository: IUserRepository = {
  userSingleById: vi.fn(),
  userSingleOrDefaultById: vi.fn(),
  userSingleOrDefaultByEmail: vi.fn(),
  userFindAll: vi.fn(),
  userCreate: vi.fn(),
  userUpdate: vi.fn(),
  userDelete: vi.fn(),
}

describe('UserService', () => {
  let userService: UserService

  beforeEach(() => {
    vi.clearAllMocks()
    userService = new UserService(mockUserRepository)
  })

  describe('getUserById', () => {
    it('should return the user when the user exists', async () => {
      // Arrange
      const userId = 'user-1'
      const expected: User = { id: userId, name: 'Alice', email: 'alice@example.com', role: 'member', isActive: true, createdAt: new Date() }
      vi.mocked(mockUserRepository.userSingleOrDefaultById).mockResolvedValue(expected)

      // Act
      const result = await userService.getUserById(userId)

      // Assert
      expect(result).toEqual(expected)
      expect(mockUserRepository.userSingleOrDefaultById).toHaveBeenCalledOnce()
      expect(mockUserRepository.userSingleOrDefaultById).toHaveBeenCalledWith(userId)
    })
  })
})
```

**Rules:**
- `vi.clearAllMocks()` in `beforeEach` — never share mock state between tests
- Always assert **both** the return value AND the mock call (count + arguments)
- Use `vi.mocked()` for type-safe mock access
- Avoid `vi.mock(modulePath)` for application services — prefer explicit constructor injection

### React Components — Testing Library

Test what the user sees and does, not implementation details. Do not assert on component state or internal methods.

```typescript
// user-card.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { describe, it, expect, vi } from 'vitest'
import { UserCard } from './user-card'

const fakeUser: User = {
  id: 'user-1',
  name: 'Alice',
  email: 'alice@example.com',
  role: 'member',
  isActive: true,
  createdAt: new Date(),
}

describe('UserCard', () => {
  it('renders the user name and email', () => {
    // Arrange + Act
    render(<UserCard user={fakeUser} onDelete={vi.fn()} />)

    // Assert
    expect(screen.getByText('Alice')).toBeInTheDocument()
    expect(screen.getByText('alice@example.com')).toBeInTheDocument()
  })

  describe('when the delete button is clicked', () => {
    it('calls onDelete with the user id', async () => {
      // Arrange
      const onDelete = vi.fn()
      render(<UserCard user={fakeUser} onDelete={onDelete} />)

      // Act
      fireEvent.click(screen.getByRole('button', { name: /delete/i }))

      // Assert
      expect(onDelete).toHaveBeenCalledOnce()
      expect(onDelete).toHaveBeenCalledWith('user-1')
    })
  })
})
```

**Rules:**
- Query by role, label, or visible text — never by `data-testid` unless no semantic alternative exists
- `data-testid` is a last resort, not a default
- Do not assert on CSS classes, component state, or internal props
- Wrap `ServicesProvider` with mock services for container/page components (see [react.md](react.md))

### Angular Components — TestBed

```typescript
// user-list-view.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing'
import { screen } from '@testing-library/angular'
import { render } from '@testing-library/angular'
import { UserListViewComponent } from './user-list-view.component'

const fakeUsers: User[] = [
  { id: 'user-1', name: 'Alice', email: 'alice@example.com', role: 'member', isActive: true, createdAt: new Date() },
]

describe('UserListViewComponent', () => {
  it('renders a list item for each user', async () => {
    // Arrange + Act
    await render(UserListViewComponent, {
      componentInputs: { users: fakeUsers, isLoading: false },
    })

    // Assert
    expect(screen.getByText('Alice')).toBeInTheDocument()
  })

  it('renders the empty state when users is an empty array', async () => {
    await render(UserListViewComponent, {
      componentInputs: { users: [], isLoading: false },
    })

    expect(screen.getByText(/no users/i)).toBeInTheDocument()
  })
})
```

---

## Exception / Error Testing

```typescript
it('should throw ValidationError when the user is an admin', async () => {
  // Arrange
  const adminUser: User = { ...fakeUser, role: 'admin' }
  vi.mocked(mockUserRepository.userSingleById).mockResolvedValue(adminUser)

  // Act + Assert
  await expect(userService.deleteUser(adminUser.id))
    .rejects.toThrow(ValidationError)
})
```

For `Result<T>` failures (non-throwing):

```typescript
it('should return a failure result when the email is already in use', async () => {
  // Arrange
  vi.mocked(mockUserRepository.userSingleOrDefaultByEmail).mockResolvedValue(fakeUser)

  // Act
  const result = await userService.createUser({ email: fakeUser.email, name: 'Bob', role: 'member' })

  // Assert
  expect(result.success).toBe(false)
  if (!result.success) {
    expect(result.error).toContain('already in use')
  }
})
```

---

## E2E Testing

Use **Playwright** for critical user flows. See the `e2e-runner` agent for implementation patterns.

```typescript
// e2e/users.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Users page', () => {
  test('should display the user list when users exist', async ({ page }) => {
    await page.goto('/users')
    await expect(page.getByRole('list')).toBeVisible()
  })

  test('should delete a user when the delete button is confirmed', async ({ page }) => {
    await page.goto('/users')
    await page.getByRole('button', { name: /delete alice/i }).click()
    await page.getByRole('button', { name: /confirm/i }).click()
    await expect(page.getByText('Alice')).not.toBeVisible()
  })
})
```

---

## TypeScript Testing Checklist

Before committing tests:

- [ ] Outer `describe` matches the class or component name exactly
- [ ] Inner `describe` names the method (services) or user action/state (components)
- [ ] Every `it` string starts with `'should'` and reads as a complete sentence
- [ ] No vague names: `it('works')`, `it('handles error')`, `it('test 1')` are forbidden
- [ ] `vi.clearAllMocks()` / `jest.clearAllMocks()` called in `beforeEach`
- [ ] Services: mock constructed explicitly via interface — no `vi.mock` module patching
- [ ] Components: queried by role/label/text — no `data-testid` unless unavoidable
- [ ] Every test asserts both return value AND mock call (count + arguments) for service tests
- [ ] AAA pattern with labelled comments
- [ ] Exception tests use `.rejects.toThrow()` — not try/catch
- [ ] `Result<T>` failures asserted on `result.success === false` and `result.error` content
- [ ] React page/container tests wrap with `<ServicesProvider services={mockServices}>`
- [ ] 80%+ coverage maintained
- [ ] TDD workflow followed — test written before implementation
