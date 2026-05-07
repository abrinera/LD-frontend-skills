---
name: tanstack-migration
description: >
  Full migration guide for React apps moving from React Router DOM + Supabase to TanStack Router + TanStack Query + Axios REST API. Use this skill whenever the user wants to migrate routing from react-router-dom to TanStack Router, replace Supabase with a REST API, set up TanStack Query for data fetching, wire up Axios with auth interceptors, or replace Supabase auth with token-based auth. Also triggers when user says "integrate different API", "switch to tanstack", "remove supabase", "set up react query", or asks how to wire TanStack Router with TanStack Query in an existing project. Always use this skill before writing any migration code.
---
 
# TanStack Migration Skill
 
Full migration from **React Router DOM + Supabase** → **TanStack Router + TanStack Query + Axios**.
 
## Scope
 
- Replace `react-router-dom` with `@tanstack/react-router`
- Replace Supabase client with Axios `apiClient`
- Replace Supabase auth with token-based auth
- Set up `@tanstack/react-query` for all data fetching
- Provide file-by-file import swap reference
---
 
## Step 1 — Install / Remove Deps
 
```bash
npm install @tanstack/react-router @tanstack/react-query @tanstack/react-query-devtools axios
npm remove @supabase/supabase-js react-router-dom
```
 
Also remove if present:
```bash
npm remove next-themes   # only if used solely for Supabase/sonner theming
```
 
---
 
## Step 2 — Axios API Client
 
**`src/lib/apiClient.ts`**
 
```ts
import axios from "axios";
 
export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_BASE_URI,
  headers: { "Content-Type": "application/json" },
});
 
// Attach token to every request
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
 
// Global 401 → redirect to login
apiClient.interceptors.response.use(
  (res) => res,
  (err) => {
    if (err.response?.status === 401) {
      localStorage.removeItem("token");
      window.location.href = "/login";
    }
    return Promise.reject(err);
  }
);
```
 
> `VITE_BASE_URI` must exist in `.env.dev` / `.env.uat` / `.env.prod`. Never commit secrets.
 
---
 
## Step 3 — Auth Hooks
 
**`src/hooks/useAuth.ts`**
 
```ts
import { useMutation } from "@tanstack/react-query";
import { apiClient } from "@/lib/apiClient";
 
export const useLogin = () =>
  useMutation({
    mutationFn: async (creds: { email: string; password: string }) => {
      const { data } = await apiClient.post("/auth/login", creds);
      localStorage.setItem("token", data.token);
      return data;
    },
  });
 
export const useLogout = () => () => {
  localStorage.removeItem("token");
  window.location.href = "/login";
};
 
export const isAuthenticated = () => !!localStorage.getItem("token");
```
 
> Replace all `supabase.auth.*` calls with these hooks.
 
---
 
## Step 4 — TanStack Router Setup
 
**`src/router.tsx`**
 
```tsx
import {
  createRouter,
  createRoute,
  createRootRoute,
  redirect,
} from "@tanstack/react-router";
import { isAuthenticated } from "@/hooks/useAuth";
import RootLayout from "@/components/layout/RootLayout";
import LoginPage from "@/pages/LoginPage";
import DashboardPage from "@/pages/DashboardPage";
 
const rootRoute = createRootRoute({ component: RootLayout });
 
const loginRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: "/login",
  component: LoginPage,
});
 
const dashboardRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: "/",
  beforeLoad: () => {
    if (!isAuthenticated()) throw redirect({ to: "/login" });
  },
  component: DashboardPage,
});
 
// Add more routes here following same pattern
const routeTree = rootRoute.addChildren([loginRoute, dashboardRoute]);
 
export const router = createRouter({ routeTree });
 
// Required for TypeScript type-safety
declare module "@tanstack/react-router" {
  interface Register { router: typeof router }
}
```
 
### Adding new routes
 
```tsx
const workspacesRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: "/workspaces",
  beforeLoad: () => {
    if (!isAuthenticated()) throw redirect({ to: "/login" });
  },
  component: WorkspacesPage,
});
// Add to routeTree: rootRoute.addChildren([..., workspacesRoute])
```
 
---
 
## Step 5 — Root Layout
 
**`src/components/layout/RootLayout.tsx`**
 
```tsx
import { Outlet } from "@tanstack/react-router";
import Sidebar from "./Sidebar";
 
export default function RootLayout() {
  return (
    <div className="flex h-screen">
      <Sidebar />
      <main className="flex-1 overflow-auto">
        <Outlet />
      </main>
    </div>
  );
}
```
 
---
 
## Step 6 — Wire Everything in main.tsx
 
**`src/main.tsx`**
 
```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import { RouterProvider } from "@tanstack/react-router";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { router } from "./router";
import "./index.css";
 
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: 1, staleTime: 1000 * 60 * 5 },
  },
});
 
ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <RouterProvider router={router} />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  </React.StrictMode>
);
```
 
> `QueryClientProvider` wraps `RouterProvider` so queries work inside routes.
 
---
 
## Step 7 — Data Fetching Pattern
 
**`src/hooks/use[Resource].ts`** — template for any resource:
 
```ts
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { apiClient } from "@/lib/apiClient";
 
// READ
export const useWorkspaces = () =>
  useQuery({
    queryKey: ["workspaces"],
    queryFn: async () => {
      const { data } = await apiClient.get("/workspaces");
      return data;
    },
  });
 
// CREATE
export const useCreateWorkspace = () => {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async (payload: { name: string }) => {
      const { data } = await apiClient.post("/workspaces", payload);
      return data;
    },
    onSuccess: () => qc.invalidateQueries({ queryKey: ["workspaces"] }),
  });
};
 
// UPDATE / DELETE follow same pattern
```
 
---
 
## Import Swap Reference
 
| Old import | New import |
|---|---|
| `from "react-router-dom"` | `from "@tanstack/react-router"` |
| `useNavigate` | `useNavigate` (same name, TanStack) |
| `Link` | `Link` (same name, TanStack) |
| `Outlet` | `Outlet` (same name, TanStack) |
| `useLocation` | `useLocation` (same name, TanStack) |
| `supabase.from(...).select()` | `apiClient.get("/endpoint")` |
| `supabase.auth.signInWithPassword()` | `useLogin()` mutation |
| `supabase.auth.signOut()` | `useLogout()()` |
| `@/integrations/supabase/client` | `@/lib/apiClient` |
| `@/contexts/AuthContext` | `@/hooks/useAuth` |
 
---
 
## Cleanup Checklist
 
- [ ] Delete `src/integrations/supabase/`
- [ ] Delete `src/contexts/AuthContext.tsx`
- [ ] Delete `src/Provider/AuthContext.tsx` (if duplicate)
- [ ] Replace `NavigationContext` usage with `useRouter` / `useNavigate` from TanStack
- [ ] Remove Supabase env vars from all `.env.*` files
- [ ] Verify `VITE_BASE_URI` set correctly per environment
- [ ] Remove `@/hooks/use-toast` — use `sonner` directly: `import { toast } from "sonner"`
---
 
## Common Gotchas
 
**QueryClient must wrap RouterProvider** — not the other way around. Routes need query access.
 
**`beforeLoad` for auth guards** — use `throw redirect(...)` not `return redirect(...)`.
 
**TanStack Router is file-route or code-route** — this skill uses code-based routes. File-based routes require Vite plugin setup (`@tanstack/router-plugin`).
 
**`staleTime`** — set per query or globally. Default 0 = refetch on every mount. Set `1000 * 60 * 5` (5 min) for stable data.
 
**Token storage** — `localStorage` is fine for most SPAs. For high-security apps, consider `httpOnly` cookies + a `/auth/refresh` endpoint.