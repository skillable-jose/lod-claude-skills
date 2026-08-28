---
paths:
  - "**/*.tsx"
  - "**/*.ts"
  - "src/**/*.tsx"
  - "src/**/*.ts"
---
# TypeScript - React Clean Architecture

> React-specific implementation of the clean architecture layers defined in [frontend-arch.md](frontend-arch.md). Read that file first for layer definitions, domain types, repository and service rules. See [css.md](css.md) for CSS architecture: design tokens, CSS Modules scoping rules, ITCSS structure, and theming.

---

## Repository Implementation in React

Repositories are plain classes — no React imports. Inject the base URL (or an `axios` instance) via the constructor.

```typescript
// repositories/users/http-user-repository.ts
export class HttpUserRepository implements IUserRepository {
  constructor(private readonly http: ApiClient) {}

  async userSingleById(id: string): Promise<User> {
    const user = await this.userSingleOrDefaultById(id)
    if (!user) throw new NotFoundException(`User not found (UserId: ${id})`)
    return user
  }

  async userSingleOrDefaultById(id: string): Promise<User | null> {
    const res = await this.http.get<UserDto>(`/users/${id}`)
    return res ? this.mapToDomain(res) : null
  }

  private mapToDomain(dto: UserDto): User { ... }
}
```

A lightweight `ApiClient` wrapping `fetch` (or `axios`) belongs in `core/api-client.ts`. It handles auth headers, base URL, and throws `AppError` on non-2xx responses — keeping individual repositories clean.

---

## Service Implementation in React

Services are plain classes. No React hooks, no `useState`, no `useEffect`.

```typescript
// services/users/user-service.ts
export class UserService implements IUserService {
  constructor(private readonly userRepo: IUserRepository) {}

  // ... same as frontend-arch.md examples
}
```

---

## Composition Root

Wire all dependencies in `core/providers.tsx`. This is the React equivalent of `Program.cs`.

```typescript
// core/providers.tsx
import { createContext, useContext, type ReactNode } from 'react'
import { HttpUserRepository } from '../repositories/http-user-repository'
import { UserService } from '../services/user-service'
import { ApiClient } from './api-client'

interface Services {
  userService: IUserService
}

const apiClient = new ApiClient(import.meta.env.VITE_API_BASE_URL)
const userRepository = new HttpUserRepository(apiClient)
const defaultServices: Services = {
  userService: new UserService(userRepository),
}

const ServicesContext = createContext<Services>(defaultServices)

export function ServicesProvider({ children, services = defaultServices }: {
  children: ReactNode
  services?: Services
}) {
  return <ServicesContext.Provider value={services}>{children}</ServicesContext.Provider>
}

export const useServices = () => useContext(ServicesContext)
```

```typescript
// main.tsx
root.render(
  <ServicesProvider>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </ServicesProvider>
)
```

**Testing**: Wrap the component under test with `<ServicesProvider services={mockServices}>` — no `vi.mock` or global `fetch` patching needed.

```typescript
// test example
const mockServices = {
  userService: {
    getActiveUsers: vi.fn().mockResolvedValue([fakeUser]),
    deleteUser: vi.fn().mockResolvedValue(undefined),
  } satisfies IUserService
}

render(
  <ServicesProvider services={mockServices}>
    <UsersPage />
  </ServicesProvider>
)
```

---

## State Layer: React Query + Zustand

React Query manages **server state** (fetching, caching, background refresh, optimistic updates). Zustand manages **client/UI state** (selected items, open panels, theme).

### Query Hooks (server state)

One hook per query use case. The hook is the "controller" — it calls the service, handles loading/error states, and returns typed data. No business logic lives here.

```typescript
// state/users/use-user-list.ts
import { useQuery } from '@tanstack/react-query'
import { useServices } from '../../core/providers'

export function useUserList() {
  const { userService } = useServices()

  return useQuery({
    queryKey: ['users', 'active'],
    queryFn: () => userService.getActiveUsers(),
  })
}
```

```typescript
// state/users/use-user-by-id.ts
export function useUserById(id: string) {
  const { userService } = useServices()

  return useQuery({
    queryKey: ['users', id],
    queryFn: () => userService.getUserById(id),
    enabled: !!id,
  })
}
```

### Mutation Hooks (commands)

```typescript
// state/users/use-create-user.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'

export function useCreateUser() {
  const { userService } = useServices()
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (data: CreateUserRequest) => userService.createUser(data),
    onSuccess: (result) => {
      if (result.success) {
        queryClient.invalidateQueries({ queryKey: ['users'] })
      }
    },
  })
}
```

```typescript
// state/users/use-delete-user.ts
export function useDeleteUser() {
  const { userService } = useServices()
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (id: string) => userService.deleteUser(id),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['users'] }),
  })
}
```

### Client State: Zustand

Use Zustand for UI state that is not server-derived: selected rows, open modals, sidebar state.

```typescript
// state/users/user-ui-store.ts
import { create } from 'zustand'

interface UserUiState {
  selectedUserId: string | null
  isDeleteDialogOpen: boolean
  selectUser: (id: string) => void
  openDeleteDialog: () => void
  closeDeleteDialog: () => void
}

export const useUserUiStore = create<UserUiState>((set) => ({
  selectedUserId: null,
  isDeleteDialogOpen: false,
  selectUser: (id) => set({ selectedUserId: id }),
  openDeleteDialog: () => set({ isDeleteDialogOpen: true }),
  closeDeleteDialog: () => set({ isDeleteDialogOpen: false, selectedUserId: null }),
}))
```

**Rule:** Do NOT store server data (users, orders) in Zustand. React Query is the source of truth for server state. Zustand holds only UI interaction state.

---

## Smart Components (Containers)

Smart components compose hooks and pass data to presentational components. They should contain minimal JSX.

```typescript
// pages/users/users-page.tsx
export function UsersPage() {
  const { data: users = [], isLoading, error } = useUserList()
  const { mutate: deleteUser, isPending: isDeleting } = useDeleteUser()
  const { selectedUserId, selectUser, isDeleteDialogOpen, openDeleteDialog, closeDeleteDialog } = useUserUiStore()

  function handleDeleteRequest(id: string) {
    selectUser(id)
    openDeleteDialog()
  }

  function handleDeleteConfirm() {
    if (selectedUserId) deleteUser(selectedUserId)
    closeDeleteDialog()
  }

  return (
    <UserListView
      users={users}
      isLoading={isLoading}
      error={error?.message}
      onDeleteRequest={handleDeleteRequest}
      deleteDialog={
        <ConfirmDialog
          open={isDeleteDialogOpen}
          isLoading={isDeleting}
          onConfirm={handleDeleteConfirm}
          onCancel={closeDeleteDialog}
        />
      }
    />
  )
}
```

---

## Presentational Components

Receive all data via props. No `useQuery`, no `useServices`, no Zustand. Local UI state (`useState`, `useRef`) is allowed.

```typescript
// components/users/user-list-view/user-list-view.tsx
interface UserListViewProps {
  users: readonly User[]
  isLoading: boolean
  error?: string
  onDeleteRequest: (id: string) => void
  deleteDialog?: ReactNode
}

export function UserListView({ users, isLoading, error, onDeleteRequest, deleteDialog }: UserListViewProps) {
  if (isLoading) return <Spinner />
  if (error) return <ErrorMessage message={error} />

  return (
    <>
      <ul>
        {users.map(user => (
          <UserCard key={user.id} user={user} onDelete={() => onDeleteRequest(user.id)} />
        ))}
      </ul>
      {deleteDialog}
    </>
  )
}
```

```typescript
// components/users/user-card/user-card.tsx
interface UserCardProps {
  user: User
  onDelete: () => void
}

export function UserCard({ user, onDelete }: UserCardProps) {
  return (
    <li>
      <span>{user.name}</span>
      <span>{user.email}</span>
      <button onClick={onDelete}>Delete</button>
    </li>
  )
}
```

---

## React Layer-First File Structure

```
src/
  state/
    users/
      use-user-list.ts
      use-user-by-id.ts
      use-create-user.ts
      use-delete-user.ts
      user-ui-store.ts
  components/
    users/
      user-card/
        user-card.tsx
        user-card.test.tsx
        user-card.module.css
      user-list-view/
        user-list-view.tsx
        user-list-view.test.tsx
      user-form/
        user-form.tsx
        user-form.test.tsx
    shared/
      ...
  pages/
    users/
      users-page.tsx
      user-edit-page.tsx
  core/
    providers.tsx
    api-client.ts
```

---

## React Clean Architecture Checklist

Before committing React feature code:
- [ ] No `fetch`/`axios` calls outside of `repositories/`
- [ ] No business logic inside hooks — hooks call service methods only
- [ ] No service injection inside presentational components
- [ ] Smart components: minimal JSX, primarily hook composition
- [ ] Presentational components: all data via props, no `useQuery`/`useServices`
- [ ] `Result<T>` returned by service mutations is handled at the mutation hook level
- [ ] Zustand stores hold UI state only — no server data cached there
- [ ] Tests use `<ServicesProvider services={mockServices}>` — no global fetch mocking
- [ ] Pages import from `state/` and `components/` within the same feature scope — no cross-feature component imports
- [ ] Domain types imported from `domain/` — no inline type definitions that duplicate domain models
