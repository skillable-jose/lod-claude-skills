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

> This file extends [common/testing.md](../common/testing.md) with TypeScript/JavaScript-specific conventions. Read that file first for the intent rule, TDD workflow, AAA pattern, and coverage requirements. See [frontend-arch.md](frontend-arch.md) for the feature-folder architecture these examples assume.

---

## Test Framework & Tools

| Concern | React (Vite SPA) — org baseline | Angular |
|---|---|---|
| Unit/integration runner | **Vitest** | **Jest** (default) or Vitest |
| Component rendering | **React Testing Library** | **Angular Testing Library** / `TestBed` |
| Integration (network-level) | **MSW** — mock at the network layer, not the feature's `api/` module | — |
| Mocking | `vi.fn()` / `vi.spyOn()` / `vi.mock()` | `jest.fn()` / `jest.spyOn()` |
| E2E | **Playwright** | **Playwright** |
| Assertions | Vitest `expect` + Testing Library matchers | Jest `expect` + Testing Library matchers |

Vitest is the org's recorded baseline for React — trust it by default. Still, **verify against the target repo's actual `package.json` `test` script before assuming**: a repo can have `vitest` sitting unused in `devDependencies` from a stalled migration while its `test` script actually runs `jest` (or vice versa). If the real script disagrees with this table, the real script wins — note the discrepancy once, then proceed with what's real.

Jest's `describe`/`it`/`expect`/`jest` globals are ambient — real Jest test files typically have no test-framework import at all. Vitest requires the explicit `import { describe, it, expect, vi } from 'vitest'` shown in the examples below.

---

## Test Naming Convention

TypeScript tests do not use method names — they use nested `describe`/`it` string blocks. The three-part intent rule from [common/testing.md](../common/testing.md) maps to the nesting structure:

| Intent part | Maps to |
|---|---|
| What is under test | Outer `describe` — module or component name |
| Function or behaviour under test | Inner `describe` — function name or user action |
| Outcome + condition | `it('should ... when ...')` string |

### Feature `api/` Module Tests

```typescript
// features/users/api/users.api.test.ts
describe('users.api', () => {
  describe('getUserById', () => {
    it('should return the user when the request succeeds', async () => { ... })
    it('should return a failure ApiResponse when the user does not exist', async () => { ... })
    it('should return a failure ApiResponse when the network request fails', async () => { ... })
  })

  describe('createUser', () => {
    it('should return the created user when the request is valid', async () => { ... })
    it('should return a failure ApiResponse when the email is already in use', async () => { ... })
  })
})
```

### React Component Tests — User Perspective

React component tests express behaviour from the **user's perspective**, not the implementation's. The inner `describe` names a user action or rendered state, not a function name.

```typescript
// UserCard.test.tsx
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

Angular component tests follow the same describe/it structure. Use `TestBed` for component tests and plain class instantiation for service tests. Angular isn't governed by the React feature-folder ADR — see [angular.md](angular.md) for its own architecture.

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
- Outer `describe` = module or component name — always matches the file's subject exactly
- Inner `describe` = function name (api/hooks) or user action / rendered state (components)
- `it` string starts with `'should'` and reads as a complete sentence
- Vague names are forbidden: `it('works')`, `it('handles error')`, `it('test 1')` — all forbidden
- The full intent must be readable from `describe` + `it` without opening the test body

---

## File Co-location

Tests live alongside the source file they test — not in a separate `__tests__` folder (React feature folders):

```
features/users/
  api/
    users.api.ts
    users.api.test.ts
  hooks/
    use-user-list.ts
    use-user-list.test.ts
  components/
    UserCard.tsx
    UserCard.test.tsx
  pages/
    UsersPage.tsx
    UsersPage.test.tsx
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
it('should return the user when the request succeeds', async () => {
  // Arrange
  const userId = 'user-1'
  const expectedUser: User = { id: userId, name: 'Alice', email: 'alice@example.com' }
  vi.mocked(client.get).mockResolvedValue({ success: true, data: expectedUser })

  // Act
  const result = await getUserById(userId)

  // Assert
  expect(result).toEqual({ success: true, data: expectedUser })
  expect(client.get).toHaveBeenCalledOnce()
  expect(client.get).toHaveBeenCalledWith(`/users/${userId}`)
})
```

---

## Mocking

### Feature `api/` Modules — Mock the Shared Client

This is the org's baseline shape (per [frontend-arch.md](frontend-arch.md)): a feature's `api/` module is a set of plain functions calling the one shared `shared/api/client.ts`. There is no constructor to inject a fake into, so mocking the shared client module is the correct, expected tool — not a workaround.

```typescript
// features/users/api/users.api.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { client } from '../../../shared/api/client'
import { getUserById, createUser } from './users.api'

vi.mock('../../../shared/api/client')

describe('users.api', () => {
  beforeEach(() => vi.clearAllMocks())

  describe('getUserById', () => {
    it('should return the user when the request succeeds', async () => {
      // Arrange
      const userId = 'user-1'
      const expected: User = { id: userId, name: 'Alice', email: 'alice@example.com' }
      vi.mocked(client.get).mockResolvedValue({ success: true, data: expected })

      // Act
      const result = await getUserById(userId)

      // Assert
      expect(result).toEqual({ success: true, data: expected })
      expect(client.get).toHaveBeenCalledOnce()
      expect(client.get).toHaveBeenCalledWith(`/users/${userId}`)
    })
  })
})
```

**Rules:**
- `vi.clearAllMocks()` / `jest.clearAllMocks()` in `beforeEach` — never share mock state between tests
- Always assert **both** the return value AND the mock call (count + arguments)
- Use `vi.mocked()` / `jest.mocked()` for type-safe mock access
- Mock `shared/api/client.ts`, not `global.fetch` — the client is the actual integration seam per `frontend-arch.md` Rule 2

**If a project deliberately uses a class with constructor-injected dependencies** (not the org baseline, but a legitimate choice some projects make): construct the mock dependency object explicitly and inject it instead of module-mocking the class.

### React Components — Testing Library

Test what the user sees and does, not implementation details. Do not assert on component state or internal methods.

```typescript
// UserCard.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { describe, it, expect, vi } from 'vitest'
import { UserCard } from './UserCard'

const fakeUser: User = { id: 'user-1', name: 'Alice', email: 'alice@example.com' }

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
- Wrap with whatever the project's real provider setup is (`AuthProvider`, `ThemeProvider` from `app/providers/`) — never invent a `ServicesProvider`; there is no composition root/DI container in this architecture

### Angular Components — TestBed

```typescript
// user-list-view.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing'
import { screen } from '@testing-library/angular'
import { render } from '@testing-library/angular'
import { UserListViewComponent } from './user-list-view.component'

const fakeUsers: User[] = [
  { id: 'user-1', name: 'Alice', email: 'alice@example.com' },
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

## Integration Testing with MSW

For a hook or page that exercises multiple `api/` calls together, mock at the network layer with MSW instead of mocking `shared/api/client.ts` directly — this exercises the real client code (auth headers, error translation) against a fake server:

```typescript
// features/users/hooks/use-user-list.integration.test.ts
import { setupServer } from 'msw/node'
import { http, HttpResponse } from 'msw'
import { describe, it, expect, beforeAll, afterEach, afterAll } from 'vitest'
import { renderHook, waitFor } from '@testing-library/react'
import { useUserList } from './use-user-list'

const server = setupServer(
  http.get('/api/users', () => HttpResponse.json({ success: true, data: [{ id: '1', name: 'Alice' }] }))
)

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())

describe('useUserList', () => {
  it('should populate users when the API call succeeds', async () => {
    const { result } = renderHook(() => useUserList())

    await waitFor(() => expect(result.current.isLoading).toBe(false))

    expect(result.current.users).toEqual([{ id: '1', name: 'Alice' }])
  })
})
```

---

## `ApiResponse<T>` / Error Testing

The frontend's failure contract is `ApiResponse<T>` (see [frontend-arch.md](frontend-arch.md)), not the backend's `Result<T>`, and it's a return value, not a thrown exception — never wrap it in `.rejects.toThrow()`:

```typescript
it('should return a failure ApiResponse when the email is already in use', async () => {
  // Arrange
  vi.mocked(client.post).mockResolvedValue({
    success: false,
    error: { code: 'user.duplicate_email', message: 'Email already exists' },
  })

  // Act
  const result = await createUser({ email: 'alice@example.com', name: 'Alice' })

  // Assert
  expect(result.success).toBe(false)
  if (!result.success) {
    expect(result.error.code).toBe('user.duplicate_email')
  }
})
```

For a genuinely thrown error (e.g. a hook that re-throws on an unexpected failure):

```typescript
it('should throw when the user is not authenticated', async () => {
  // Arrange
  vi.mocked(client.get).mockRejectedValue(new UnauthorizedError())

  // Act + Assert
  await expect(getUserById('user-1')).rejects.toThrow(UnauthorizedError)
})
```

---

## E2E Testing

Use **Playwright** for critical user flows.

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

- [ ] Outer `describe` matches the module or component name exactly
- [ ] Inner `describe` names the function (api/hooks) or user action/state (components)
- [ ] Every `it` string starts with `'should'` and reads as a complete sentence
- [ ] No vague names: `it('works')`, `it('handles error')`, `it('test 1')` are forbidden
- [ ] `vi.clearAllMocks()` / `jest.clearAllMocks()` called in `beforeEach`
- [ ] Feature `api/` module tests mock `shared/api/client.ts` — not `global.fetch`, not the module under test itself
- [ ] Components: queried by role/label/text — no `data-testid` unless unavoidable
- [ ] Every test asserts both return value AND mock call (count + arguments) for `api/` module tests
- [ ] AAA pattern with labelled comments
- [ ] `ApiResponse<T>` failures asserted on `result.success === false` and `result.error.code` — never wrapped in `.rejects`
- [ ] Genuinely thrown errors use `.rejects.toThrow()`
- [ ] No invented `ServicesProvider`/composition-root wrapper in component tests — use the project's real providers
- [ ] 80%+ coverage maintained (or the repo's own configured `coverageThreshold`, if different)
- [ ] TDD workflow followed — test written before implementation
