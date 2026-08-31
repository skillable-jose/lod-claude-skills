---
paths:
  - "**/*.ts"
  - "**/*.tsx"
---
# React Frontend Architecture

> This is the org's actual, ADR-mandated React/TypeScript architecture — a **feature-folder** structure, not a hand-invented layered (domain/repository/service) pattern. Normative source: the frontend-architecture ADR (registered as **ADR-AA015** — see the citation note below), read in full alongside `_shared/resolve-design-archetype.md` and `apply-design-tokens` if `ai-enablement` is installed. See [react.md](react.md) for React-specific implementation detail (hooks, component patterns) and [css.md](css.md) for styling/token mechanics.
>
> **Citation note:** every `ai-enablement` skill cites this decision as `ADR-AA015`. In the `engineering-decisions` repo as of this writing, that ID actually resolves to an unrelated draft ADR ("Durable Save/Resume for Active Hyper-V Lab Profiles") — a hygiene bug in the org's own ADR repo, not something to route around here. The real content lives in a loose, unprefixed file: `adrs/ADR_Frontend Architecture for React-TypeScript SPAs.md` (Status: Accepted; the file's own History section says it was "re-registered as ADR-AA015" but was never actually renamed to match). Treat that loose file as ground truth until the org fixes the ID collision; cite it as "ADR-AA015 (frontend architecture)" for consistency with `ai-enablement`'s tooling, but don't be surprised if a direct ID lookup misses.
>
> **Angular is a separate track.** This file (and the ADR behind it) covers React only. Angular's structure is described independently in [angular.md](angular.md) and isn't governed by this ADR — don't assume the two must match.

---

## Baseline Decision Table

| Concern | Decision |
|---|---|
| Framework | **Vite SPA**. Next.js was considered and rejected: *"Next.js instead of Vite: rejected for v1. Most prototypes are SPAs; Vite scaffolds are simpler. Revisit if SSR becomes a real need."* Treat any existing Next.js app as a pre-ADR legacy deviation, not a pattern to keep scaffolding. |
| Structure | **Feature folders** — see below. |
| Server state | `useState` in a hook by default. **TanStack Query** only when server-state complexity (caching, background refresh, optimistic updates) actually justifies it — not a default dependency. |
| Client state | `useState`/lift/Context by default. **Zustand** only when state genuinely crosses feature boundaries. |
| Styling | **Tailwind CSS**, tokens generated from the project's `DESIGN.md`. Styled-components/Emotion were considered and rejected ("Tailwind pairs better with the DESIGN.md → variables.css pipeline"). |
| Component library | **MUI or Ant Design** — either is sanctioned per project (record the choice in that project's own `ADR-0000`); never mix both in the same project. A custom Tailwind/Radix design system was proposed and **rejected**. |
| Testing | **Vitest + Testing Library** (unit), **Vitest + MSW** (integration, mock at the network layer), **Playwright** (E2E, Tier 2+). |
| API error shape | `ApiResponse<T>` discriminated union (below) — **not** the backend's `Result<T>` (ADR-IDP001). The two never share code; only the "log once, at the layer that detects" principle carries over. |

Optional dependencies (TanStack Query, Zustand) are opt-in per project, not baseline scaffolding — adding one is itself a decision worth a line in that project's `ADR-0000`.

---

## Folder Structure

```
frontend/
├── DESIGN.md
├── src/
│   ├── app/
│   │   ├── main.tsx
│   │   ├── App.tsx
│   │   ├── router.tsx
│   │   └── providers/
│   │       ├── AuthProvider.tsx
│   │       └── ThemeProvider.tsx
│   ├── features/
│   │   └── <feature-name>/
│   │       ├── api/
│   │       │   └── <feature>.api.ts
│   │       ├── components/
│   │       ├── hooks/
│   │       ├── types/
│   │       ├── pages/
│   │       └── index.ts
│   ├── shared/
│   │   ├── api/
│   │   │   ├── client.ts
│   │   │   ├── api-response.types.ts
│   │   │   └── error-handling.ts
│   │   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── feature-flags/
│   └── styles/
│       ├── variables.css
│       └── globals.css
├── public/
├── package.json
├── tsconfig.json
├── vite.config.ts
└── tailwind.config.ts
```

| Folder | Responsibility |
|---|---|
| `app/` | Bootstrap, routing, top-level providers (auth, theme). Not a feature. |
| `features/<name>/api/` | Feature-specific API calls — **plain functions**, no class, no DI container. |
| `features/<name>/components/` | Feature-scoped presentational/interactive components. |
| `features/<name>/hooks/` | Feature-scoped hooks — where server data is fetched and held (see below). |
| `features/<name>/types/` | Feature-scoped types. |
| `features/<name>/pages/` | Route-level components for this feature. |
| `features/<name>/index.ts` | **The feature's public surface.** This is the only thing outside code may import from a feature — everything else inside the folder is private to it. |
| `shared/` | Cross-feature primitives: the one HTTP client, auth utilities, feature-flag client, shared components/hooks. |

## Architectural Rules

**Rule 1 — No cross-feature imports.** Features cannot import from sibling features. Cross-feature sharing happens through `shared/` only. A feature folder must be independently movable or deletable without breaking another feature.

**Rule 2 — No `fetch()` outside `shared/api/client.ts`.** Components, hooks, and pages never call `fetch()` directly. All HTTP goes through the shared client via a feature's `<feature>.api.ts`. The shared client is where auth tokens are attached, errors are translated, and logging hooks fire — per the "log once, at the layer that detects" principle carried over from ADR-IDP001.

**Rule 3 — Server data lives in a hook, held in local state by default.** For most feature work: `useExamples()` fetches via the feature's `api/` module and holds the result in `useState`, returning it to the calling component. Reach for TanStack Query only when that gets genuinely painful (cache invalidation across components, background refetch, optimistic updates) — and record that choice.

**Rule 4 — Client state defaults to `useState`/Context.** Reach for Zustand only when the state must cross feature boundaries and prop-drilling has become the actual problem — not preemptively.

**Rule 5 — No composition root, no DI container.** There is exactly one shared HTTP client instance (`shared/api/client.ts`), imported directly by feature `api/` modules. Auth state comes from `useAuth()` (exposing `{ user, isAuthenticated, login, logout, getToken }`, provided by `app/providers/AuthProvider.tsx`); the HTTP client attaches the token automatically. Do not build a `ServicesProvider`/service-locator pattern — it doesn't exist in this architecture.

**Rule 6 — No hardcoded design values.** Colors, fonts, and spacing in component styles always reference a CSS variable generated from the project's `DESIGN.md` — never a literal hex/px value. See [css.md](css.md).

## `ApiResponse<T>` — the Frontend's Error/Result Shape

This is the contract the shared HTTP client returns and every feature `api/` module propagates — the frontend analog of the backend's `Result<T>` (ADR-IDP001), but a distinct type with no shared code between them:

```typescript
// shared/api/api-response.types.ts
export type ApiResponse<T> =
  | { success: true; data: T }
  | { success: false; error: { code: string; message: string; details?: unknown } }

export type PaginatedApiResponse<T> =
  | {
      success: true
      data: T[]
      pagination: { page: number; pageSize: number; totalCount: number; hasMore: boolean }
    }
  | { success: false; error: { code: string; message: string; details?: unknown } }
```

- Match on `response.success`, never on a thrown exception, for an expected API failure — the same "the system did its job and the answer was no" reservation rule ADR-IDP001 applies to the backend applies here too.
- Failure `code` is the stable contract element clients match on — never `message`.
- These types live in a shared package (`@skillable/api-types` in the org's actual setup) so frontend and backend agree on shape without hand-copying it per project.

## Design Tokens

Token *values* are not authored per-project by hand. They come from the org's `skillable-design-system` repo, consumed via the `apply-design-tokens` skill (if `ai-enablement` is installed) or the project's own `DESIGN.md` → `variables.css` pipeline otherwise. This file and [css.md](css.md) govern architecture and usage rules, not the specific token values — never hand-roll a competing token set.

## Testing

```
- Unit: Vitest + Testing Library, colocated with the file under test (or in __tests__/)
- Integration: Vitest + MSW — mock the API at the network layer, not by mocking the feature's api/ module
- E2E (Tier 2+): Playwright, tests in e2e/, run in CI against a deployed dev environment
```

See [testing.md](testing.md) for naming conventions and assertion patterns.

## Frontend Architecture Checklist

Before committing feature code:
- [ ] No feature imports from a sibling feature — only from `shared/`
- [ ] No `fetch()` outside `shared/api/client.ts`
- [ ] Feature's public surface is `index.ts` — nothing else is imported from outside the feature
- [ ] Server data fetched in a hook; `useState` unless TanStack Query is a recorded, justified choice
- [ ] Client state is `useState`/Context unless Zustand crosses feature boundaries and is a recorded choice
- [ ] No hand-invented `ServicesProvider`/composition-root/DI container
- [ ] API failures matched on `ApiResponse<T>.success`, never wrapped in try/catch for an expected failure
- [ ] No hardcoded color/font/spacing values — all reference a token from `DESIGN.md`/`variables.css`
- [ ] Next.js is not being introduced as a "better" alternative — Vite SPA is the baseline
