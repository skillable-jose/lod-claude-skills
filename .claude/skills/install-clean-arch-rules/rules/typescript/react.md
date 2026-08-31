---
paths:
  - "**/*.tsx"
  - "**/*.ts"
  - "src/**/*.tsx"
  - "src/**/*.ts"
---
# TypeScript - React (Vite SPA)

> React-specific implementation of the feature-folder architecture defined in [frontend-arch.md](frontend-arch.md) — read that file first for the folder structure, the five architectural rules, and the `ApiResponse<T>` shape. See [css.md](css.md) for CSS architecture and [testing.md](testing.md) for the Vitest/Testing Library/MSW conventions.

---

## Feature `api/` Modules — Plain Functions, No Class

A feature's API calls are plain exported functions, not a repository class. They call the one shared HTTP client — never `fetch` directly.

```typescript
// features/users/api/users.api.ts
import { client } from '../../../shared/api/client'
import type { ApiResponse, PaginatedApiResponse } from '../../../shared/api/api-response.types'
import type { User, CreateUserRequest } from '../types/user'

export function getActiveUsers(): Promise<PaginatedApiResponse<User>> {
  return client.get('/users', { params: { isActive: true } })
}

export function getUserById(id: string): Promise<ApiResponse<User>> {
  return client.get(`/users/${id}`)
}

export function createUser(data: CreateUserRequest): Promise<ApiResponse<User>> {
  return client.post('/users', data)
}

export function deleteUser(id: string): Promise<ApiResponse<void>> {
  return client.delete(`/users/${id}`)
}
```

`shared/api/client.ts` is the only place that attaches auth headers, translates transport errors into `ApiResponse` failures, and fires logging hooks. Individual feature `api/` modules stay thin — one function per operation, no business logic.

---

## Hooks — Where Server Data Lives

Per `frontend-arch.md` Rule 3: fetch in a hook, hold the result in `useState` by default. No composition root, no `useServices()` — the hook imports the feature's `api/` module directly.

```typescript
// features/users/hooks/use-user-list.ts
import { useEffect, useState } from 'react'
import { getActiveUsers } from '../api/users.api'
import type { User } from '../types/user'

export function useUserList() {
  const [users, setUsers] = useState<User[]>([])
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    let cancelled = false
    setIsLoading(true)

    getActiveUsers().then((response) => {
      if (cancelled) return
      if (response.success) {
        setUsers(response.data)
        setError(null)
      } else {
        setError(response.error.message)
      }
      setIsLoading(false)
    })

    return () => { cancelled = true }
  }, [])

  return { users, isLoading, error }
}
```

**Only reach for TanStack Query** when this hand-rolled pattern is genuinely insufficient — cache invalidation across multiple components, background refetch, optimistic updates — and record the decision in the project's `ADR-0000`:

```typescript
// features/users/hooks/use-user-list.ts — TanStack Query variant, once justified
import { useQuery } from '@tanstack/react-query'
import { getActiveUsers } from '../api/users.api'

export function useUserList() {
  return useQuery({
    queryKey: ['users', 'active'],
    queryFn: async () => {
      const response = await getActiveUsers()
      if (!response.success) throw new Error(response.error.message)
      return response.data
    },
  })
}
```

```typescript
// features/users/hooks/use-create-user.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { createUser } from '../api/users.api'

export function useCreateUser() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: createUser,
    onSuccess: (response) => {
      if (response.success) queryClient.invalidateQueries({ queryKey: ['users'] })
    },
  })
}
```

---

## Client State: `useState`/Context First, Zustand When Justified

```typescript
// features/users/hooks/use-user-selection.ts — Context is enough for most cases
import { createContext, useContext, useState, type ReactNode } from 'react'

interface UserSelectionState {
  selectedUserId: string | null
  selectUser: (id: string) => void
  clearSelection: () => void
}

const UserSelectionContext = createContext<UserSelectionState | null>(null)

export function UserSelectionProvider({ children }: { children: ReactNode }) {
  const [selectedUserId, setSelectedUserId] = useState<string | null>(null)
  return (
    <UserSelectionContext.Provider value={{
      selectedUserId,
      selectUser: setSelectedUserId,
      clearSelection: () => setSelectedUserId(null),
    }}>
      {children}
    </UserSelectionContext.Provider>
  )
}

export const useUserSelection = () => {
  const ctx = useContext(UserSelectionContext)
  if (!ctx) throw new Error('useUserSelection must be used within UserSelectionProvider')
  return ctx
}
```

Reach for Zustand only once state genuinely needs to cross feature boundaries and prop-drilling/Context nesting has become the actual problem — not as a default:

```typescript
// shared/hooks/use-ui-store.ts — cross-feature UI state, once justified
import { create } from 'zustand'

interface UiState {
  sidebarOpen: boolean
  toggleSidebar: () => void
}

export const useUiStore = create<UiState>((set) => ({
  sidebarOpen: true,
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
}))
```

**Rule:** Never store server data in Zustand — server data lives in a hook (`useState` or TanStack Query), Zustand holds only cross-feature UI state.

---

## Pages and Components

Pages compose hooks and pass data to components. Components stay focused on rendering and user interaction.

```typescript
// features/users/pages/users-page.tsx
import { useUserList } from '../hooks/use-user-list'
import { useCreateUser } from '../hooks/use-create-user'
import { UserListView } from '../components/UserListView'

export function UsersPage() {
  const { users, isLoading, error } = useUserList()
  const { mutate: createUser, isPending } = useCreateUser()

  return (
    <UserListView
      users={users}
      isLoading={isLoading}
      error={error}
      onCreateUser={createUser}
      isCreating={isPending}
    />
  )
}
```

```typescript
// features/users/components/UserListView.tsx
import type { User } from '../types/user'

interface UserListViewProps {
  users: readonly User[]
  isLoading: boolean
  error: string | null
  onCreateUser: (data: { name: string; email: string }) => void
  isCreating: boolean
}

export function UserListView({ users, isLoading, error, onCreateUser, isCreating }: UserListViewProps) {
  if (isLoading) return <Spinner />
  if (error) return <ErrorMessage message={error} />

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name} — {user.email}</li>
      ))}
    </ul>
  )
}
```

---

## Testing

Vitest + Testing Library, colocated with the file under test. See [testing.md](testing.md) for the full naming and assertion conventions — summary:

```typescript
// features/users/api/users.api.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { client } from '../../../shared/api/client'
import { getActiveUsers } from './users.api'

vi.mock('../../../shared/api/client')

describe('users.api', () => {
  beforeEach(() => vi.clearAllMocks())

  describe('getActiveUsers', () => {
    it('should return the active users when the request succeeds', async () => {
      vi.mocked(client.get).mockResolvedValue({ success: true, data: [], pagination: { page: 1, pageSize: 25, totalCount: 0, hasMore: false } })

      const result = await getActiveUsers()

      expect(result.success).toBe(true)
      expect(client.get).toHaveBeenCalledWith('/users', { params: { isActive: true } })
    })
  })
})
```

```typescript
// features/users/components/UserListView.test.tsx
import { render, screen } from '@testing-library/react'
import { describe, it, expect, vi } from 'vitest'
import { UserListView } from './UserListView'

describe('UserListView', () => {
  it('renders a list item for each user', () => {
    render(
      <UserListView
        users={[{ id: '1', name: 'Alice', email: 'alice@example.com' }]}
        isLoading={false}
        error={null}
        onCreateUser={vi.fn()}
        isCreating={false}
      />
    )

    expect(screen.getByText('Alice — alice@example.com')).toBeInTheDocument()
  })
})
```

**Rules:**
- Mock the shared client (`shared/api/client.ts`) with `vi.mock`, not `fetch` globally and not the feature's own `api/` module — the client is the actual integration seam.
- Query by role, label, or visible text — never `data-testid` unless there's no semantic alternative.
- Assert what the user sees/does — never component state or internal props.

---

## React (Vite SPA) Checklist

Before committing React feature code:
- [ ] No `fetch`/`axios` calls outside `shared/api/client.ts`
- [ ] Feature `api/` modules are plain functions, not classes — no repository/service layer invented
- [ ] Server data fetched in a hook; `useState` unless TanStack Query is a recorded, justified choice
- [ ] No `ServicesProvider`/composition-root pattern — there is no DI container in this architecture
- [ ] Zustand (if used) holds only cross-feature UI state, never server data
- [ ] Pages import from the same feature's `hooks/`/`components/` — no cross-feature component imports
- [ ] `ApiResponse<T>.success` checked directly — never wrapped in try/catch for an expected failure
- [ ] Tests mock `shared/api/client.ts`, not `fetch` globally
