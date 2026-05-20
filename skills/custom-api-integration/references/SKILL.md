---
name: custom-api-integration
description: >
  Add, audit, or plan API integrations in the bef-lease-drop frontend codebase. This skill enforces the project's exact architecture: feature-sliced modules under src/features/, server functions via createServerFn, backendRequest from src/server/api-client.server.ts, two-layer Zod schemas, TanStack Query hooks with key factories, and file-based TanStack Router routes under src/routes/_authed/. Trigger this skill whenever the user wants to add a new feature with API calls, audit how an existing feature integrates with the backend, write an implementation plan for a new or partial feature, scaffold query/mutation/schema/route files, or fix incorrect API patterns (wrong Axios instance, inlined schemas, missing query key factory, mutations not invalidating cache). Also trigger when the user says "add API", "integrate backend", "write a service", "scaffold a feature", "audit API integration", "implementation plan", or "how should I fetch X".
---

# Custom API Integration

Use this skill for all API integration work in the `bef-lease-drop` frontend.
The project uses a strict **feature-sliced architecture** where every domain module
owns its queries, mutations, schemas, server functions, and components.

> **Reference baseline:** `src/features/workspaces/` is the gold-standard feature.
> Always compare new or audited features against it.
> See `references/patterns.md` for the full annotated pattern card.

---

## Codebase At a Glance

```
src/
  app/              ← QueryClient setup (query-client.ts)
  config/           ← axios-config.tsx (legacy — do NOT use for feature data calls)
  dto/              ← Legacy shared DTOs (settings/, user/) — prefer feature schemas
  features/         ← ONE folder per domain — this is where all new work lives
  lib/              ← utils.ts, redirect.ts
  routes/           ← File-based TanStack Router
    __root.tsx
    _authed.tsx     ← Auth guard layout — all protected routes nest here
    _main.tsx       ← Public layout (signin, signup)
    _authed/        ← Protected route files
    _main/          ← Public route files
    analysis.tsx    ← Layout route for analysis
    analysis/       ← Dynamic analysis routes
  server/           ← Server-only layer (NEVER import in client components)
    api-client.server.ts   ← backendRequest — the ONLY HTTP client for feature calls
    auth-middleware.server.ts
    env.server.ts
  services/         ← Legacy service layer (do not extend; migrate toward features/)
  shared/           ← Shared UI, hooks, layouts, utilities
  styles/
```

---

## Feature Module Structure

Every feature under `src/features/<feature>/` must follow this layout:

```
features/<feature>/
  api/
    <feature>.actions.ts    ← createServerFn wrappers (client-callable)
    <feature>.server.ts     ← backendRequest HTTP calls (server-only)
  component/
    <FeatureComponent>.tsx  ← Scoped UI components (drawers, cards, dialogs)
  <feature>-query-keys.ts   ← Query key factory — one source of truth
  <feature>-queries.ts      ← useQuery hooks
  <feature>-mutations.ts    ← useMutation hooks
  <feature>-schemas.ts      ← Zod schemas (raw backend + transform → clean type)
  <feature>-screen.tsx      ← Top-level page component consumed by the route
```

---

## The Two-Layer API Pattern

This codebase uses **server functions** — not direct Axios calls from components.
This keeps tokens server-side and never exposes auth to the client bundle.

### Layer 1 — Server Function (`api/<feature>.actions.ts`)

```ts
import { createServerFn } from '@tanstack/react-start'
import { z } from 'zod'
import { readAuthSession } from '@/server/auth-middleware.server'
import { fetchFeatureImpl } from './feature.server'

export const fetchFeatureAction = createServerFn({ method: 'GET' })
  .inputValidator(z.object({ workspaceId: z.string() }))
  .handler(async ({ data: { workspaceId } }) => {
    const session = await readAuthSession()
    if (!session) throw new Error('Unauthorized')
    return fetchFeatureImpl(session.accessToken, workspaceId)
  })
```

**Rules:**
- Validate all inputs inline with `z.object()`
- Read session server-side with `readAuthSession()`
- Never pass raw tokens to the client
- Delegate HTTP work to the `*Impl` function in `*.server.ts`

### Layer 2 — HTTP Implementation (`api/<feature>.server.ts`)

```ts
import { backendRequest } from '@/server/api-client.server'
import { FeatureBackendResponseSchema, type Feature } from '../<feature>-schemas'

export async function fetchFeatureImpl(
  accessToken: string,
  workspaceId: string,
): Promise<Feature[]> {
  const response = await backendRequest<unknown>({
    method: 'GET',
    url: `/workspaces/${workspaceId}/feature`,
    headers: { Authorization: `Bearer ${accessToken}` },
  })
  return FeatureBackendResponseSchema.parse(response)
}
```

**Rules:**
- Use `backendRequest` from `@/server/api-client.server` — NEVER `axios-config.tsx`
- Parse + transform the response with the feature's Zod schema
- Return a typed result — never return `unknown` or raw response objects
- File must stay server-only (`.server.ts` suffix enforces this)

---

## Zod Schema Pattern (`<feature>-schemas.ts`)

Two layers — raw backend shape and a clean transformed type:

```ts
import { z } from 'zod'

// Layer 1: raw API response (snake_case, Spring envelope fields)
const FeatureBackendSchema = z.object({
  id: z.string(),
  workspace_id: z.string(),
  name: z.string(),
  status: z.string(),
  created_at: z.string(),
})

// Layer 2: paginated envelope + transform to clean camelCase type
export const FeatureBackendResponseSchema = z.object({
  content: z.array(FeatureBackendSchema),
  totalElements: z.number(),
  totalPages: z.number(),
}).transform(({ content, totalElements, totalPages }) => ({
  items: content.map((item) => ({
    id: item.id,
    workspaceId: item.workspace_id,
    name: item.name,
    status: item.status.toLowerCase() as 'active' | 'inactive',
    createdAt: item.created_at,
  })),
  totalElements,
  totalPages,
}))

// Exported clean type
export type Feature = z.infer<typeof FeatureBackendSchema>

// Form validation schema (used with react-hook-form + zodResolver)
export const createFeatureSchema = z.object({
  name: z.string().min(1, 'Name is required').max(100),
  workspaceId: z.string().min(1, 'Workspace is required'),
})
export type CreateFeatureInput = z.infer<typeof createFeatureSchema>
```

---

## Query Key Factory (`<feature>-query-keys.ts`)

```ts
export const featureQueryKeys = {
  all: ['feature'] as const,
  list: (workspaceId: string) =>
    [...featureQueryKeys.all, 'list', workspaceId] as const,
  detail: (id: string) =>
    [...featureQueryKeys.all, 'detail', id] as const,
}
```

**Rules:**
- Always use the factory in `useQuery` and `invalidateQueries` — never hardcode strings
- `all` is the root scope for broad invalidation after destructive mutations
- `list(param)` scopes paginated queries to a parent entity
- `detail(id)` for single-item fetches

---

## Query Hooks (`<feature>-queries.ts`)

```ts
import { useQuery, keepPreviousData } from '@tanstack/react-query'
import { featureQueryKeys } from './<feature>-query-keys'
import { fetchFeatureAction } from './api/<feature>.actions'

export function useFeatureQuery(workspaceId: string | undefined) {
  return useQuery({
    queryKey: featureQueryKeys.list(workspaceId ?? ''),
    queryFn: () => fetchFeatureAction({ data: { workspaceId } }),
    enabled: !!workspaceId,
    placeholderData: keepPreviousData,
  })
}
```

**Rules:**
- `enabled` guard on any nullable dependency
- `keepPreviousData` for list queries to prevent flash of empty state
- `queryFn` calls the server action — never `backendRequest` directly from client
- No `staleTime` override unless there's a specific reason

---

## Mutation Hooks (`<feature>-mutations.ts`)

```ts
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { featureQueryKeys } from './<feature>-query-keys'
import { createFeatureAction } from './api/<feature>.actions'

export function useCreateFeature() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (input: CreateFeatureInput) =>
      createFeatureAction({ data: input }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: featureQueryKeys.all })
    },
  })
}

export function useDeleteFeature() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (id: string) => deleteFeatureAction({ data: { id } }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: featureQueryKeys.all })
    },
  })
}
```

**Rules:**
- Always invalidate `featureQueryKeys.all` in `onSuccess` as the minimum
- Invalidate a narrower key if the mutation only affects a specific list scope
- Use `useMutation` — do NOT use manual async functions for new features
- Surface errors via toast (sonner) in the component's `onError`, not in the hook

---

## Screen Component (`<feature>-screen.tsx`)

```tsx
import { useFeatureQuery } from './<feature>-queries'
import { useCreateFeature } from './<feature>-mutations'
import { Skeleton } from '@/shared/ui/skeleton'

interface Props {
  workspaceId: string
}

export function FeatureScreen({ workspaceId }: Props) {
  const { data, isPending, isError } = useFeatureQuery(workspaceId)
  const createFeature = useCreateFeature()

  if (isPending) return <Skeleton className="h-64 w-full" />
  if (isError) return <p>Failed to load. Please try again.</p>
  if (!data?.items.length) return <p>No items found.</p>

  return (
    <div>
      {data.items.map((item) => (
        <div key={item.id}>{item.name}</div>
      ))}
    </div>
  )
}
```

**Rules:**
- Handle `isPending`, `isError`, and empty state explicitly — no silent failures
- Use `Skeleton` from `@/shared/ui/skeleton` for loading states
- Keep API logic in hooks — screens only consume hooks and render
- Navigate with `useNavigate` / `Link` from `@tanstack/react-router`

---

## Form Components (in `component/`)

```tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { createFeatureSchema, type CreateFeatureInput } from '../<feature>-schemas'
import { useCreateFeature } from '../<feature>-mutations'
import { toast } from 'sonner'

export function CreateFeatureDrawer() {
  const form = useForm<CreateFeatureInput>({
    resolver: zodResolver(createFeatureSchema),
    defaultValues: { name: '', workspaceId: '' },
  })
  const createFeature = useCreateFeature()

  function onSubmit(values: CreateFeatureInput) {
    createFeature.mutate(values, {
      onSuccess: () => {
        toast.success('Created successfully.')
        form.reset()
      },
      onError: () => {
        toast.error('Something went wrong. Please try again.')
      },
    })
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* fields */}
      <button type="submit" disabled={createFeature.isPending}>
        {createFeature.isPending ? 'Creating…' : 'Create'}
      </button>
    </form>
  )
}
```

**Rules:**
- Always use `react-hook-form` + `zodResolver` — never uncontrolled forms
- Schema comes from `<feature>-schemas.ts` — never inline Zod in a component
- Error messages come from toast (sonner) — never `alert()` or `console.error`
- Disable submit button while `isPending`

---

## Route File (`src/routes/_authed/<feature>.tsx`)

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { FeatureScreen } from '@/features/<feature>/<feature>-screen'

export const Route = createFileRoute('/_authed/<feature>')({
  component: FeatureScreen,
})
```

**Rules:**
- All authenticated routes go under `src/routes/_authed/`
- The `_authed.tsx` layout enforces the auth guard — do not re-implement it
- Dynamic segments use `$param` convention (e.g. `$workspaceId.tsx`)
- Public routes go under `src/routes/_main/`

---

## Known Anti-Patterns — Never Replicate

| Anti-pattern | Where it exists | What to do instead |
|---|---|---|
| `src/config/axios-config.tsx` used for feature calls | Legacy services/ files | Use `backendRequest` via server function |
| Manual `Authorization` header in components | Some drawer components | Token is attached in `backendRequest` server-side |
| Zod schema inlined inside a component | Some older drawers | Move to `<feature>-schemas.ts` |
| Mutations as raw `async` functions without `useMutation` | `workspaceView.tsx` drawers | Use `useMutation` with `onSuccess` invalidation |
| `console.log` in server auth paths | `auth.server.ts` line 189/199 | Remove — never log tokens or session data |
| `dummy-login.tsx` with `DEV_BEARER_TOKEN` | `src/services/` | Must not ship — remove before prod build |
| `workspaces-screen.tsx` entirely commented out | `features/workspaces/` | Delete — it is dead code |
| `.tsx` extension on non-JSX service files | `src/services/api/` | Use `.ts` for files with no JSX |
| Typo `componenets/` directory | `features/analysis/` | Correct to `components/` when touching that feature |

---

## Implementation Plan Template

When asked to plan a new feature, produce this structure:

### 1. Classify the feature
- **Greenfield** — no files exist → scaffold everything
- **Partial** — some files exist → audit gaps, plan additions
- **Existing** — full implementation → audit against workspaces baseline

### 2. Gap table (Partial / Existing only)

| Area | Expected | Current | Gap | Severity |
|---|---|---|---|---|
| Query key factory | `<feature>-query-keys.ts` | | | |
| Query hooks | `<feature>-queries.ts` | | | |
| Mutation hooks | `<feature>-mutations.ts` | | | |
| Zod schemas | `<feature>-schemas.ts` | | | |
| Server action | `api/<feature>.actions.ts` | | | |
| HTTP impl | `api/<feature>.server.ts` | | | |
| Screen | `<feature>-screen.tsx` | | | |
| Route | `routes/_authed/<feature>.tsx` | | | |

### 3. Ordered implementation steps

1. **Schemas** — `src/features/<feature>/<feature>-schemas.ts`
2. **Query key factory** — `src/features/<feature>/<feature>-query-keys.ts`
3. **HTTP impl** — `src/features/<feature>/api/<feature>.server.ts`
4. **Server actions** — `src/features/<feature>/api/<feature>.actions.ts`
5. **Query hooks** — `src/features/<feature>/<feature>-queries.ts`
6. **Mutation hooks** — `src/features/<feature>/<feature>-mutations.ts`
7. **Components** — `src/features/<feature>/component/`
8. **Screen** — `src/features/<feature>/<feature>-screen.tsx`
9. **Route** — `src/routes/_authed/<feature>.tsx`
10. **Cleanup** — dead code, wrong extensions, inlined schemas *(skip if greenfield)*

### 4. Confidence check
Answer before writing any code:
1. Any file with unclear purpose that affects the plan?
2. Any existing pattern that contradicts the workspaces baseline — match or correct?
3. Any endpoints not yet confirmed to exist in the backend?

> **Do not write implementation code until the plan is approved.**

---

## Checklist — New Feature

- [ ] Schemas defined in `<feature>-schemas.ts` (raw + transform + form schema)
- [ ] Query key factory in `<feature>-query-keys.ts`
- [ ] HTTP impl in `api/<feature>.server.ts` using `backendRequest`
- [ ] Server function in `api/<feature>.actions.ts` using `createServerFn`
- [ ] `useQuery` hook in `<feature>-queries.ts` with `enabled` guard
- [ ] `useMutation` hooks in `<feature>-mutations.ts` with `onSuccess` invalidation
- [ ] Forms use `react-hook-form` + `zodResolver(schema)`
- [ ] Loading / error / empty states handled in screen
- [ ] Route created under `src/routes/_authed/`
- [ ] No tokens, secrets, or `console.log` of sensitive data
- [ ] No `axios-config.tsx` used for feature data calls
- [ ] No Zod schemas inlined in components

For deeper pattern examples and annotated snippets, read `references/patterns.md`.