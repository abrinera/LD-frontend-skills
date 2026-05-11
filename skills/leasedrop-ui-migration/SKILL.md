---
name: leasedrop-ui-migration
description: >
  Selective UI migration from leasedrop-lovable (SOURCE) into bef-lease-drop (TARGET).
  User specifies source page + destination route. Only UI markup, Tailwind classes, and
  custom visual components are copied. TARGET's logic layer (services, auth, DTOs, routes,
  env vars) is never modified. Triggers: "migrate UI from lovable", "copy page design",
  "port the UI for <feature>", "migrate <page> UI".
---

# LeaseDrop Selective UI Migration Skill

> **Scope:** Visual design, layout, and component styling only.
> **Never touch:** API services, auth, env config, Zod schemas, or data-fetching hooks.

## Paths

```
SOURCE: D:\Work projects\lease-drop-migration\leasedrop-lovable
TARGET: D:\Work projects\lease-drop-migration\bef-lease-drop
```

---

## Key Differences (Always Remember)

| Concern | SOURCE | TARGET |
|---|---|---|
| Tailwind | v3 + `tailwindcss-animate` + `tailwind.config.ts` | v4 + `tw-animate-css` + `@theme inline` in CSS |
| CSS vars | HSL (`230 99% 48%`) | OKLCH + HSL in `data-theme` blocks |
| Router | `react-router-dom` v6 | TanStack Router v1 (file-based) |
| Auth | Supabase | Custom `AuthContext` + Axios interceptors |
| Data | Supabase + mock | Axios REST API + TanStack Query |
| shadcn import | `@/components/ui/` | `@/shadcn/components/ui/` |
| Layout | Sidebar-based | Header-based (`AppLayout`) |
| Icons | `lucide-react` | `lucide-react` + `iconsax-react` |
| Form fields | raw shadcn inputs | custom `FormInput`, `FormSelect` etc from `@/custome-ui/ui/user/form/` |

---

## Migration Protocol

### Step 0 — Receive User Input

User provides:
```
Source: <source page file> + <source component folder>
Destination: <target route path>
```

### Step 1 — Read SOURCE files

1. Read the source page file completely
2. Read all sub-component files it imports from `src/components/<feature>/`
3. List every:
   - shadcn/ui import
   - lucide-react icon import
   - Custom component import
   - Data/hook dependency (to strip or rewire)
   - Router hook usage (to adapt)

### Step 2 — Prepare TARGET

1. Check if destination route exists. If not, create route file in `src/routes/`
2. Check if page component exists. If not, create page wrapper in `src/view/pages/<feature>/`
3. Identify which shadcn components the source page uses that TARGET lacks. Install them.
4. Identify any missing icon imports.

### Step 3 — Adapt and Copy

For each SOURCE component:

#### 3a — Import Rewiring

| SOURCE import | TARGET replacement |
|---|---|
| `from "@/components/ui/<x>"` | `from "@/shadcn/components/ui/<x>"` |
| `from "@/lib/utils"` | `from "@/shadcn/lib/utils"` |
| `from "react-router-dom"` (Link, useNavigate, useLocation, useParams, useSearchParams) | `from "@tanstack/react-router"` (Link, useNavigate, useLocation, useParams, useSearch) |
| `useNavigate()` then `navigate("/path")` | `useNavigate()` then `navigate({ to: "/path" })` |
| `useParams()` | `Route.useParams()` or `useParams({ from: "/route/$param" })` |
| `useSearchParams()` | `useSearch({ from: "/route" })` or `Route.useSearch()` |
| `from "@/contexts/AuthContext"` | `from "@/config/auth/hooks/useAuth"` |
| `useAuth()` | `useContext(AuthContext)` |
| `from "@/contexts/NavigationContext"` | Strip — wire through props or TARGET equivalent |
| Supabase imports | Strip entirely — accept data via props |

#### 3b — Data Stripping

- Remove all `supabase.from(...)` calls
- Remove all inline `useQuery`/`useMutation` that call Supabase
- Replace with props interface or TARGET's existing service hooks
- Keep mock data if user requested it (copy to `src/constant/mock/`)

#### 3c — Component Placement

| Component type | TARGET path |
|---|---|
| Page wrapper | `src/view/pages/<feature>/<Feature>Page.tsx` |
| Form components | `src/view/pages/<feature>/forms/<Feature>Form.tsx` |
| Feature sub-components | `src/custome-ui/components/leasedrop/<feature>/` |
| Shared UI atoms | `src/custome-ui/elements/leasedrop/` |
| Mock data | `src/constant/mock/` |

#### 3d — Route Registration

```tsx
// src/routes/_authenticated/<feature>/route.tsx
import { createFileRoute } from "@tanstack/react-router";
import FeaturePage from "@/view/pages/<feature>/<Feature>Page.tsx";

export const Route = createFileRoute("/_authenticated/<feature>")({
  component: () => <FeaturePage />,
});
```

### Step 4 — Verify

```bash
# Build check
npm run build

# No hardcoded secrets
grep -rn -E "(api_key|secret|password|token)\s*[:=]\s*['\"][^'\"]{8,}" src/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Auth guard intact
grep -rn "isAuthenticated\|beforeLoad" src/routes/_authenticated/ --include="*.tsx"

# API client untouched
grep -rn "createAxiosInstance" src/config/ --include="*.tsx"
```

Visual check in browser — compare SOURCE and TARGET side-by-side.

---

## SOURCE Feature Catalog

| # | Feature | Source Files | Sub-components |
|---|---|---|---|
| 1 | Landing | `pages/Landing.tsx` | `components/landing/*` (9 files) |
| 2 | Auth | `pages/Auth.tsx` | `components/auth/*` (2 files) |
| 3 | Command Centre | `pages/CommandCentre.tsx` | `components/command/*` (6 files) |
| 4 | Workspaces | `pages/Workspaces.tsx` | `components/workspaces/*` (7 files) |
| 5 | Documents | `pages/Documents.tsx` | `components/upload/*` (10 files) |
| 6 | Lease Dashboard | `pages/LeaseDashboard.tsx` | `components/lease-dashboard/*` (11 files) |
| 7 | Lease Results | `pages/LeaseResults.tsx` | `components/lease-results/*` (4 files) |
| 8 | Tasks | `pages/Tasks.tsx` | `components/tasks/*` (2 files) |
| 9 | People & Access | `pages/PeopleAccess.tsx` | `components/people/*` (3 files) |
| 10 | Analytics | `pages/Analytics.tsx` | (inline, uses recharts) |
| 11 | Correspondence | `pages/Correspondence.tsx` | `components/correspondence/*` (3 files) |
| 12 | Properties | `pages/Properties.tsx` | — |
| 13 | Key Dates | `pages/KeyDates.tsx` | — |
| 14 | Workflows | `pages/Workflows.tsx` | — |
| 15 | Notices | `pages/Notices.tsx` | — |
| 16 | Agents | `pages/Agents.tsx` | — |
| 17 | Consultants | `pages/Consultants.tsx` | — |
| 18 | Lease Intelligence | `pages/LeaseIntelligence.tsx` | — |
| 19 | Onboarding | `pages/Onboarding.tsx` | — |
| 20 | Account | `pages/Account.tsx` | `components/account/*` (5 files) |
| 21 | Billing | `pages/SubscriptionBilling.tsx` | — |
| 22 | Audit | `pages/AuditCompliance.tsx` | `components/audit/*` (2 files) |
| 23 | Blog | `pages/Blog.tsx`, `BlogPost.tsx` | — |
| 24 | Trust | `pages/Trust.tsx` | — |
| 25 | Social Impact | `pages/SocialImpact.tsx` | — |
| 26 | Invite Flow | `pages/InviteFlow.tsx` | `components/invite/*` (8 files) |
| 27 | Sidebar | — | `components/layout/*` (7 files) |
| 28 | Leases | `pages/Leases.tsx` | — |
| 29 | Integrations | `pages/Integrations.tsx` | — |

---

## Files This Skill Touches

| File | Action |
|---|---|
| `src/index.css` | Add theme tokens (once) |
| `index.html` | Add font imports (once) |
| `src/routes/_authenticated/<feature>/*.tsx` | Create route files |
| `src/view/pages/<feature>/` | Create/update page components |
| `src/custome-ui/components/leasedrop/<feature>/` | Add custom components |
| `src/custome-ui/elements/leasedrop/` | Add shared UI atoms |
| `src/constant/mock/` | Add mock data (optional) |

## Files This Skill NEVER Touches

| File | Reason |
|---|---|
| `src/config/axios-config.tsx` | API wiring |
| `src/config/auth/` | Auth logic |
| `src/services/` | Service hooks |
| `src/dto/` | Zod schemas |
| `src/routes/__root.tsx` | Root config |
| `src/routes/_authenticated/route.tsx` | Auth guard |
| `.env*` | Secrets |
| `src/shadcn/components/ui/` | Only add, never modify existing |
