---
paths:
  - "**/*.tsx"
  - "**/*.ts"
  - "app/**/*.tsx"
  - "app/**/*.ts"
---
# TypeScript - Next.js (App Router) Clean Architecture

> Next.js-specific implementation of the clean architecture layers defined in [frontend-arch.md](frontend-arch.md). Read that file first for layer definitions, domain types, repository and service rules. See [css.md](css.md) for CSS architecture and [react.md](react.md) for the Client Component patterns (hooks, React Query, Zustand) that apply unchanged inside `'use client'` components.

---

## Deployment Model

A Next.js frontend is a **standalone Node.js application**, not a `.esproj`-hosted SPA. It is not added to the `.sln` and is not built with `dotnet build`. It calls the backend as a separate HTTP service — pair it with `[ProjectNamespace].Web.Api`, not `Web.Server`. See `csharp/modularization.md` → "Next.js Frontend" and `csharp/scaffolding.md` → "Next.js Frontend (standalone)" for the backend-side wiring (CORS, base URL configuration).

---

## Server Components vs. Client Components

Every component is a Server Component by default. Add `'use client'` at the top of a file only when the component needs interactivity (state, effects, event handlers, browser-only APIs) or a library that requires the browser (React Query, Zustand).

| Concern | Server Component | Client Component (`'use client'`) |
|---|---|---|
| Data fetching | Calls services directly, `await`ed inline | Uses React Query hooks from `state/` |
| State/hooks | Not allowed | `useState`, `useEffect`, Zustand allowed |
| Access to secrets/server env vars | Yes | No — never import server-only modules here |
| Where it lives | `app/**/page.tsx`, `app/**/layout.tsx` by default | Presentational leaf components needing interactivity, smart components composing hooks |

**Rule:** Push `'use client'` as far down the tree as possible. A page can be a Server Component that renders a small Client Component island for the interactive part — it does not need to make the whole page a Client Component.

---

## Repository & Service Layers

Identical to [frontend-arch.md](frontend-arch.md) — plain classes, no framework imports, no `'use client'`. These run on both the server (imported by Server Components) and the client (imported by Client Component hooks), so they must not assume either environment.

```typescript
// repositories/users/http-user-repository.ts — same as react.md, unchanged
export class HttpUserRepository implements IUserRepository {
  constructor(private readonly http: ApiClient) {}
  // ...
}
```

**`ApiClient` base URL rule:**
- Server Components and Route Handlers call the backend using the **internal/server URL** (e.g., `API_BASE_URL`, a plain server-only env var — no `NEXT_PUBLIC_` prefix).
- Client Components call the backend using the **public URL** (e.g., `NEXT_PUBLIC_API_BASE_URL` — the only kind of env var accessible in browser code).
- Never read a `NEXT_PUBLIC_*` var expecting it to carry a secret — anything with that prefix is inlined into the client bundle at build time.

---

## Composition Root

Split the composition root in two, because Server Components and Client Components run in different environments and cannot share a module-level singleton safely across requests.

```typescript
// core/services.server.ts — Server Component / Route Handler composition root
import 'server-only'
import { HttpUserRepository } from '../repositories/users/http-user-repository'
import { UserService } from '../services/users/user-service'
import { ApiClient } from './api-client'

export function getUserService(): IUserService {
  const apiClient = new ApiClient(process.env.API_BASE_URL!)
  return new UserService(new HttpUserRepository(apiClient))
}
```

```typescript
// core/providers.tsx — Client Component composition root ('use client')
'use client'
import { createContext, useContext, type ReactNode } from 'react'
import { HttpUserRepository } from '../repositories/users/http-user-repository'
import { UserService } from '../services/users/user-service'
import { ApiClient } from './api-client'

interface Services {
  userService: IUserService
}

const apiClient = new ApiClient(process.env.NEXT_PUBLIC_API_BASE_URL!)
const defaultServices: Services = {
  userService: new UserService(new HttpUserRepository(apiClient)),
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

The `server-only` import in `services.server.ts` makes Next.js throw a build error if that module is ever accidentally imported from a Client Component — use it on every server-side composition root.

**Testing**: Same pattern as [react.md](react.md) — wrap the Client Component under test with `<ServicesProvider services={mockServices}>`. For Server Components, call the service function directly in the test (no rendering needed to test data-loading logic) or use React Testing Library's async server-component render support. Note the runner differs from react.md's Vite-SPA default: most Next.js apps run **Jest** (via `next/jest`), not Vitest — see [typescript/testing.md](testing.md#test-framework--tools). Swap `vi.fn()`/`vi.mock()` for `jest.fn()`/`jest.mock()` accordingly.

---

## Data Fetching in Server Components

Server Components call the service layer directly — no `useQuery`, no client-side fetch waterfall.

```typescript
// app/users/page.tsx (Server Component — no 'use client')
import { getUserService } from '../../core/services.server'
import { UserListView } from '../../components/users/user-list-view/user-list-view'

export default async function UsersPage() {
  const users = await getUserService().getActiveUsers()
  return <UserListView users={users} />
}
```

For interactive pieces (delete button, filters, optimistic updates), compose a small Client Component that uses React Query — same hooks and mutation patterns as [react.md](react.md) `## State Layer: React Query + Zustand`.

```typescript
// components/users/user-list-view/user-list-view.tsx ('use client' — needs the delete interaction)
'use client'
import { useDeleteUser } from '../../../state/users/use-delete-user'

interface UserListViewProps {
  users: readonly User[]  // passed down from the Server Component parent, hydrated as initial data
}

export function UserListView({ users }: UserListViewProps) {
  const { mutate: deleteUser } = useDeleteUser()
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>
          {user.name} <button onClick={() => deleteUser(user.id)}>Delete</button>
        </li>
      ))}
    </ul>
  )
}
```

**Rule:** Do not duplicate the same fetch in both a Server Component and a client-side `useQuery` on mount. Pass server-fetched data down as the query's `initialData` if the Client Component needs to refetch/mutate afterward.

---

## Route Handlers (`app/api/**/route.ts`)

Only add Route Handlers when Next.js needs to act as a thin BFF (e.g., setting an httpOnly cookie after login, aggregating two backend calls for one page). They are not the primary API — the ASP.NET Core `Web.Api` project is. Keep Route Handlers as thin adapters that call the service layer, exactly like an ASP.NET controller calls `IUserService`:

```typescript
// app/api/users/route.ts
import { getUserService } from '../../../core/services.server'
import { NextResponse } from 'next/server'

export async function GET() {
  const users = await getUserService().getActiveUsers()
  return NextResponse.json(users)
}
```

Do not put business logic, validation, or direct `fetch` calls to the backend inside a Route Handler — delegate to `services/`, same as everywhere else.

---

## Folder Structure

```
app/                           ← App Router: routes, layouts, route handlers
  users/
    page.tsx                   ← Server Component route
    [id]/
      page.tsx
  api/
    users/
      route.ts                 ← Route Handler (BFF only, optional)
  layout.tsx
domain/                        ← same as frontend-arch.md — types, interfaces, errors, Result<T>
repositories/                  ← same as frontend-arch.md — HTTP implementations
services/                      ← same as frontend-arch.md — business logic
state/                         ← Client-only: React Query hooks, Zustand stores
components/                    ← 'use client' presentational components needing interactivity
  users/
    user-card/
    user-list-view/
core/
  services.server.ts           ← Server Component / Route Handler composition root
  providers.tsx                ← Client Component composition root
  api-client.ts
```

---

## Next.js Clean Architecture Checklist

Before committing Next.js feature code:
- [ ] `'use client'` used only where interactivity or client-only libraries require it — default to Server Components
- [ ] No `NEXT_PUBLIC_*` env var carries a secret; server-only config uses unprefixed env vars behind `import 'server-only'`
- [ ] Server Components fetch via `services.server.ts`, not `fetch()`/`axios` inline and not React Query
- [ ] Client Components fetch via `state/` hooks (React Query), never a direct repository/service call in JSX
- [ ] No duplicate fetch: server-loaded data is passed down as `initialData`, not re-fetched on mount
- [ ] Route Handlers (if any) are thin — they call `services/`, they do not contain business logic
- [ ] Domain types imported from `domain/` — no inline type definitions that duplicate domain models
- [ ] `ApiClient` base URL matches its environment (`API_BASE_URL` server-side, `NEXT_PUBLIC_API_BASE_URL` client-side)
