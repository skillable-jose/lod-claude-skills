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

| Concern | React (Vite SPA) | Next.js | Angular |
|---|---|---|---|
| Unit/integration runner | **Vitest** | **Jest** (via `next/jest`) | **Jest** (default) or Vitest |
| Component rendering | **React Testing Library** | **React Testing Library** (Server Components: call the async function directly, no render needed) | **Angular Testing Library** / `TestBed` |
| Mocking | `vi.fn()` / `vi.spyOn()` | `jest.fn()` / `jest.spyOn()` / `jest.mock()` | `jest.fn()` / `jest.spyOn()` |
| E2E | **Playwright** | **Playwright** | **Playwright** |
| Assertions | Vitest `expect` + Testing Library matchers | Jest `expect` + Testing Library matchers | Jest `expect` + Testing Library matchers |

Default Next.js to **Jest** — `next/jest` gives zero-config Jest with SWC transforms and is what `create-next-app` and most existing Next.js codebases ship with. Vitest needs manual Next.js wiring and is the exception, not the default. If the target repo's `package.json` already has a `test` script, trust that over this table — e.g. a repo running `jest --maxWorkers=50%` is Jest regardless of `vitest` also sitting in `devDependencies` (a partial/abandoned migration is common; don't assume its presence means Vitest is live).

Jest's `describe`/`it`/`expect`/`jest` globals are ambient — real Jest test files typically have no test-framework import at all. Vitest requires the explicit `import { describe, it, expect, vi } from 'vitest'` shown in the examples below; drop that import and swap `vi.*` for `jest.*` when the target project is on Jest.

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
- `vi.clearAllMocks()` / `jest.clearAllMocks()` in `beforeEach` — never share mock state between tests
- Always assert **both** the return value AND the mock call (count + arguments)
- Use `vi.mocked()` / `jest.mocked()` for type-safe mock access
- Avoid `vi.mock(modulePath)` / `jest.mock(modulePath)` for application **services** (classes with constructor-injected dependencies) — prefer explicit constructor injection with a mock object, as above

**Exception — plain exported functions with no constructor to inject into:** many real codebases (especially Next.js apps that predate or skip the `frontend-arch.md` layering) expose API calls as plain functions calling `fetch` directly, not as methods on a DI'd repository class. There's nothing to construct-inject there, so module-level mocking is the correct and expected tool:

```typescript
// advisor.test.ts — module under test exports plain functions, not a class
jest.mock('../utils/api/authToken');

describe('advisor', () => {
  beforeAll(() => {
    (getAuthToken as jest.Mock).mockResolvedValue({ accessToken: TEST_ACCESS_TOKEN });
  });

  describe('getRecommendationSummary', () => {
    it('calls API with correct URL and options', async () => {
      global.fetch = jest.fn().mockResolvedValue({ ok: true, status: 200, json: async () => ({ numLabs: 0 }) });

      const result = await getRecommendationSummary(AdvisorRecommendationStatus.NEW);

      expect(result.responseData).toStrictEqual({ numLabs: 0 });
      expect(global.fetch).toHaveBeenCalledWith(expect.stringContaining('/recommendationsummary'), expect.any(Object));
    });
  });
});
```

Reach for constructor injection (no module mocking) whenever you're the one designing the module; reach for `jest.mock()`/module patching only when working inside an existing function-exports module that has no class to inject into.

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
- [ ] Services (classes with constructor-injected dependencies): mock constructed explicitly via interface — no `vi.mock`/`jest.mock` module patching. Plain exported functions with no constructor to inject into are the documented exception — module mocking is expected there.
- [ ] Components: queried by role/label/text — no `data-testid` unless unavoidable
- [ ] Every test asserts both return value AND mock call (count + arguments) for service tests
- [ ] AAA pattern with labelled comments
- [ ] Exception tests use `.rejects.toThrow()` — not try/catch
- [ ] `Result<T>` failures asserted on `result.success === false` and `result.error` content
- [ ] If the repo uses the `frontend-arch.md` layered composition root: page/container tests wrap with `<ServicesProvider services={mockServices}>`. Otherwise, use the repo's actual render helper / provider wrapper (e.g. a project-specific `renderHelper` composing its real context providers) — don't invent a `ServicesProvider` that doesn't exist in the codebase.
- [ ] 80%+ coverage maintained (or the repo's own configured `coverageThreshold`, if different)
- [ ] TDD workflow followed — test written before implementation
