# Pattern Reference — bef-lease-drop API Integration

Extended annotated examples derived from the `features/workspaces` baseline audit.
Read this file when you need deeper context on a specific pattern or edge case.

---

## Table of Contents

1. [Workspaces Pattern Card (gold standard)](#1-workspaces-pattern-card)
2. [Query Key Factory — Extended Examples](#2-query-key-factory)
3. [Two-Layer API — Annotated Deep Dive](#3-two-layer-api)
4. [Zod Schema Layers — Full Example](#4-zod-schema-layers)
5. [Pagination Pattern](#5-pagination-pattern)
6. [File Upload Pattern](#6-file-upload-pattern)
7. [Auth Feature Differences](#7-auth-feature-differences)
8. [Legacy services/ Layer — When to Touch It](#8-legacy-services-layer)
9. [Flags & Anomalies Reference](#9-flags--anomalies)

---

## 1. Workspaces Pattern Card

The gold-standard reference extracted from `src/features/workspaces/`.

```
Query Key Factory : workspacesQueryKeys.all / .list(organizationId)
                   (.detail(id) does not exist yet — add when single-item fetch needed)

useQuery shape    : { queryKey, queryFn, enabled: !!dep, placeholderData: keepPreviousData }
                   No staleTime override. No select transform (transform happens in Zod).

useMutation shape : NOT present in workspaces-mutations.ts (gap — mutations live in drawers)
                   Correct pattern is in people-access-mutations.ts:
                   useMutation({ mutationFn, onSuccess: () => queryClient.invalidateQueries({ queryKey: keys.all }) })

Zod usage         : Two-layer — FeatureBackendSchema (raw) + ResponseSchema (envelope + transform)
                   Form schemas also in <feature>-schemas.ts, used via zodResolver in components

Server fn pattern : createServerFn from @tanstack/react-start
                   Layer 1 actions.ts  → inputValidator → readAuthSession → call Impl
                   Layer 2 server.ts   → backendRequest → Zod parse → return typed data

HTTP client       : backendRequest() from src/server/api-client.server.ts
                   NEVER use src/config/axios-config.tsx for feature data calls
                   Token attached server-side only — components never touch auth headers

Form pattern      : react-hook-form + zodResolver(schema from <feature>-schemas.ts)
                   Always in component/ subdirectory, never in the screen directly

Error handling    : backendRequest throws ApiError / SessionExpiredError
                   Components handle via onError in useMutation → toast (sonner)
                   Never expose raw error objects or stack traces to users

Loading state     : <Skeleton> while isPending
                   Empty component / message when data.length === 0
                   isError → inline error message with retry suggestion

Navigation        : useNavigate / Link from @tanstack/react-router
                   Never window.location.href except in auth redirect fallback
```

---

## 2. Query Key Factory

### Minimal (list only)
```ts
export const featureQueryKeys = {
  all: ['feature'] as const,
  list: (workspaceId: string) =>
    [...featureQueryKeys.all, 'list', workspaceId] as const,
}
```

### Full (list + detail + filtered list)
```ts
export const featureQueryKeys = {
  all: ['feature'] as const,
  list: (workspaceId: string) =>
    [...featureQueryKeys.all, 'list', workspaceId] as const,
  filteredList: (workspaceId: string, filters: Record<string, unknown>) =>
    [...featureQueryKeys.list(workspaceId), JSON.stringify(filters)] as const,
  detail: (id: string) =>
    [...featureQueryKeys.all, 'detail', id] as const,
}
```

### Invalidation strategy
```ts
// Invalidate everything for this feature (after create/delete)
queryClient.invalidateQueries({ queryKey: featureQueryKeys.all })

// Invalidate only the list for a specific workspace (after update)
queryClient.invalidateQueries({ queryKey: featureQueryKeys.list(workspaceId) })

// Invalidate a single item (after patch)
queryClient.invalidateQueries({ queryKey: featureQueryKeys.detail(id) })
```

---

## 3. Two-Layer API

### Why two layers?

`createServerFn` from `@tanstack/react-start` runs on the server but is
callable from client components. This means:
- Tokens never reach the browser bundle
- Session validation happens server-side on every call
- `backendRequest` is never imported in client code

### actions.ts — What goes here

- Input validation (inline `z.object()` — not the shared feature schema)
- Session check (`readAuthSession()`)
- Delegation to `*Impl`
- Never direct HTTP calls

### server.ts — What goes here

- `backendRequest` HTTP calls
- Zod parsing of raw response
- Type-safe return value
- No session logic (that's in actions.ts)

### Mutation action example

```ts
// api/feature.actions.ts
export const createFeatureAction = createServerFn({ method: 'POST' })
  .inputValidator(z.object({
    name: z.string().min(1),
    workspaceId: z.string(),
  }))
  .handler(async ({ data }) => {
    const session = await readAuthSession()
    if (!session) throw new Error('Unauthorized')
    return createFeatureImpl(session.accessToken, data)
  })

// api/feature.server.ts
export async function createFeatureImpl(
  accessToken: string,
  input: { name: string; workspaceId: string },
): Promise<Feature> {
  const response = await backendRequest<unknown>({
    method: 'POST',
    url: `/workspaces/${input.workspaceId}/feature`,
    headers: { Authorization: `Bearer ${accessToken}` },
    data: input,
  })
  return FeatureBackendSchema.parse(response)
}
```

---

## 4. Zod Schema Layers

### Pattern: raw → transform → clean type

```ts
// 1. Raw backend shape (match API exactly, including snake_case)
const FeatureBackendSchema = z.object({
  id: z.string(),
  workspace_id: z.string(),
  display_name: z.string(),
  is_active: z.boolean(),
  created_at: z.string().datetime(),
})

// 2. Paginated list envelope + transform
export const FeatureListResponseSchema = z.object({
  content: z.array(FeatureBackendSchema),
  totalElements: z.number(),
  totalPages: z.number(),
  number: z.number(),  // current page (0-based from Spring)
}).transform(({ content, totalElements, totalPages, number }) => ({
  items: content.map((raw) => ({
    id: raw.id,
    workspaceId: raw.workspace_id,
    name: raw.display_name,
    isActive: raw.is_active,
    createdAt: new Date(raw.created_at),
  })),
  totalElements,
  totalPages,
  currentPage: number,
}))

// 3. Single item response (no envelope)
export const FeatureDetailResponseSchema = FeatureBackendSchema.transform((raw) => ({
  id: raw.id,
  workspaceId: raw.workspace_id,
  name: raw.display_name,
  isActive: raw.is_active,
  createdAt: new Date(raw.created_at),
}))

// 4. Exported clean types
export type Feature = z.infer<typeof FeatureDetailResponseSchema>
export type FeatureList = z.infer<typeof FeatureListResponseSchema>

// 5. Form schemas (used with zodResolver — not for API parsing)
export const createFeatureSchema = z.object({
  name: z.string().min(1, 'Name is required').max(100),
  workspaceId: z.string().min(1),
})
export type CreateFeatureInput = z.infer<typeof createFeatureSchema>

export const updateFeatureSchema = createFeatureSchema.partial().extend({
  id: z.string(),
})
export type UpdateFeatureInput = z.infer<typeof updateFeatureSchema>
```

---

## 5. Pagination Pattern

Spring Boot returns zero-based `number` for current page and wraps content
in a `content` array. The transform in the schema should normalize this.

```ts
// Query hook with pagination params
export function useFeatureListQuery(workspaceId: string, page = 0, size = 20) {
  return useQuery({
    queryKey: featureQueryKeys.filteredList(workspaceId, { page, size }),
    queryFn: () => fetchFeatureListAction({ data: { workspaceId, page, size } }),
    enabled: !!workspaceId,
    placeholderData: keepPreviousData,
  })
}

// In the server action input
.inputValidator(z.object({
  workspaceId: z.string(),
  page: z.number().int().min(0).default(0),
  size: z.number().int().min(1).max(100).default(20),
}))

// backendRequest with pagination
const response = await backendRequest<unknown>({
  method: 'GET',
  url: `/workspaces/${workspaceId}/feature?page=${page}&size=${size}`,
  headers: { Authorization: `Bearer ${accessToken}` },
})
```

---

## 6. File Upload Pattern

File uploads use multipart/form-data via the server layer.
Validate MIME type and size in the UI before the server action is called.

```tsx
// Component-side validation (before calling mutation)
const ALLOWED_TYPES = ['application/pdf', 'image/png', 'image/jpeg']
const MAX_SIZE_MB = 10

function validateFile(file: File): string | null {
  if (!ALLOWED_TYPES.includes(file.type)) return 'Invalid file type.'
  if (file.size > MAX_SIZE_MB * 1024 * 1024) return `File exceeds ${MAX_SIZE_MB} MB.`
  return null
}

// Server action (multipart)
export const uploadFeatureDocumentAction = createServerFn({ method: 'POST' })
  .inputValidator(z.object({ workspaceId: z.string(), fileName: z.string() }))
  .handler(async ({ data }) => {
    // Note: binary file data is passed separately, not via inputValidator
    const session = await readAuthSession()
    if (!session) throw new Error('Unauthorized')
    return uploadFeatureDocumentImpl(session.accessToken, data)
  })

// server.ts
export async function uploadFeatureDocumentImpl(
  accessToken: string,
  input: { workspaceId: string; fileName: string; file: Blob },
): Promise<{ documentId: string }> {
  const formData = new FormData()
  formData.append('file', input.file, input.fileName)
  formData.append('workspaceId', input.workspaceId)

  const response = await backendRequest<unknown>({
    method: 'POST',
    url: '/documents/upload',
    headers: {
      Authorization: `Bearer ${accessToken}`,
      'Content-Type': 'multipart/form-data',
    },
    data: formData,
  })
  return z.object({ documentId: z.string() }).parse(response)
}
```

---

## 7. Auth Feature Differences

`features/auth/` follows a slightly different pattern because authentication
is stateful and cannot use the standard `backendRequest` + session pattern
(the session doesn't exist yet during login).

Key differences:
- Uses `auth-context.ts` + `auth-provider.tsx` for session state
- `auth-mutations.ts` calls auth endpoints directly without `readAuthSession()`
- Login response is stored via the auth context, not localStorage directly
- Do NOT model new features after `features/auth/` — use `features/workspaces/` instead

---

## 8. Legacy services/ Layer — When to Touch It

`src/services/api/` contains the old service layer. Rules:

| Scenario | Action |
|---|---|
| Adding a brand new feature | Create under `src/features/` — do NOT add to `services/` |
| Fixing a bug in an existing service | Fix in place, note it as legacy in a comment |
| Migrating an existing service | Move to `src/features/<feature>/api/` + update all imports |
| `common.tsx` files or files with `BK`/date suffixes | Do NOT touch without explicit confirmation |
| `dummy-login.tsx` | Do NOT modify — flag for removal before prod |

The `src/config/axios-config.tsx` file is **orphaned** — it is not used by any
feature module and conflicts with the server function pattern. Do not use it
for new feature work.

---

## 9. Flags & Anomalies

These exist in the codebase and must not be replicated:

| File / Location | Issue | Risk |
|---|---|---|
| `src/services/dummy-login.tsx` | Contains `DEV_BEARER_TOKEN` hardcoded | **Must not ship to prod** |
| `src/features/auth/api/auth.server.ts` line 189/199 | `console.log` in prod code path | Token/session data may leak to logs |
| `src/config/axios-config.tsx` | Client-side token handling — unused by features | Creates dual-pattern confusion |
| `features/workspaces/workspaces-screen.tsx` | Entirely commented out (60 lines) | Dead code — safe to delete |
| `features/analysis/componenets/` | Directory name typo | Fix to `components/` when editing |
| `src/services/api/*.tsx` | `.tsx` on non-JSX files | Use `.ts` for new files |
| `features/workspaces/` | No `workspaces-mutations.ts` | Mutations are in drawer components — should be extracted |
| Auth token passed explicitly in `fetchWorkspacesImpl` | `backendRequest` already attaches token | Minor redundancy — do not replicate |