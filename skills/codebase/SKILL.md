---
name: codebase
description: >
  Coding conventions, patterns, and rules for the bef-QMS React frontend
  (Vite + TailwindCSS v4 + TanStack Router + React Hook Form + shadcn/ui).
  Follow this guide exactly when generating or modifying any route, page, form, or component.
---

## Codebase Summary

### Stack
| Concern | Library |
|---|---|
| Bundler | Vite 7 (`vite --mode dev/uat/prod`) |
| Framework | React 19 (StrictMode) |
| Routing | TanStack Router v1 — file-based, auto-generated `routeTree.gen.ts` |
| Server state | TanStack Query v5 (`useMutation`, `useQuery`) |
| Forms | **React Hook Form v7** + `@hookform/resolvers` + **Zod v4** |
| Styling | TailwindCSS v4 (Vite plugin, no config file) |
| UI primitives | shadcn/ui (stored in `src/shadcn/components/ui/`) |
| Icons | `lucide-react`, `iconsax-react` |
| Toasts | `sonner` (`<Toaster richColors />` in `__root.tsx`) |
| HTTP | `axios` via `createAxiosInstance` (`src/config/axios-config.tsx`) |
| Auth | Custom `AuthProvider` + `AuthContext` (`src/config/auth/`) |
| Theme | `next-themes` via `ThemeProvider` (`src/shadcn/components/ThemeProvider.tsx`) |

### Routing Tree
```
src/routes/
├── __root.tsx          ← Root: ThemeProvider + Toaster + Outlet
├── _main.tsx           ← Layout group → renders AppLayout (with footer)
│   └── index.tsx       ← "/" → HomePage
│   └── contact.tsx     ← "/contact"
│   └── legal/          ← "/legal/*"
├── _other.tsx          ← Layout group → renders NoFooterLayout (no footer)
│   └── signup.tsx      ← "/signup?planCode=..."
│   └── email-verify-failed.tsx
│   └── email-verify-success.tsx
│   └── payment-failed.tsx
│   └── payment-success.tsx
└── _info/
    └── route.tsx
```
- Routes are created with `createFileRoute(...)` (file-based convention).
- Auth-aware root uses `createRootRouteWithContext<AuthRouterContext>()`.
- Search params are validated via the `validateSearch` option on `createFileRoute`.

### Path Aliases
`@/` → `src/` (configured in `tsconfig.app.json`)

---

## Conventions

### FORMS

**Folder rule:** Every form lives in a `forms/` subfolder inside its page directory.

```
src/view/pages/<feature>/
├── <Feature>Page.tsx      ← page wrapper (scroll, layout context)
└── forms/
    └── <Feature>Form.tsx  ← all form logic lives here
```

**Form stack:**
- `useForm<Dto>({ mode: "onChange", resolver: zodResolver(schema), defaultValues })` from `react-hook-form`
- Zod schema is defined in `src/dto/<Name>Dto.ts` and exported as both schema + inferred type
- Wrap in `<FormProvider {...form}>` so custom field components can use `useFormContext`

**Form UI primitives** (from `src/shadcn/components/ui/field.tsx`):
```tsx
import { Field, FieldDescription, FieldGroup, FieldLegend, FieldSet } from "@/shadcn/components/ui/field";
```
Use `FieldGroup` as the card-like container, `FieldSet`/`FieldLegend` for title, `Field` for rows.

**Mutation pattern:**
```tsx
const { mutationName } = useXxxService();
mutationName.mutate(payload, { onSuccess: () => { ... } });
// disabled={mutationName.isPending}
```
Services live in `src/services/` and export a custom hook (`useXxxService`).

**Reusable field components** (import from `@/custome-ui/ui/user/form/basic/custom-field-shadcn/`):
| Component | Use case |
|---|---|
| `FormInput` | Text / email / number inputs |
| `FormCombobox` | Searchable select (e.g. country) |
| `FormComboboxLazy` | Server-side searchable select |
| `FormSelect` | Simple `<select>` |
| `FormTextarea` | Multi-line text |
| `FormCheckBox` | Single checkbox |
| `FormCheckBoxDetails` | Checkbox with description |
| `FormRadio` | Radio group |
| `FormCalendar` | Single date picker |
| `FormCalendarRange` | Date range picker |
| `FormSwitch` | Toggle switch |
| `FormUpload` | File upload (react-dropzone) |
| `FormFieldArray` | Dynamic field array |

### LAYOUTS

Two layout wrappers exist — choose based on whether the page needs a footer:

| Layout | File | Use when |
|---|---|---|
| `AppLayout` | `src/custome-ui/layout/AppLayout.tsx` | Standard pages (home, contact, legal) — includes `AppHeader` + `AppFooter` |
| `NoFooterLayout` | `src/custome-ui/layout/NoFooterLayout.tsx` | Transactional / standalone pages (signup, payment, email-verify) — includes `AppHeader` only |

**Rules:**
- Never add `<AppHeader />` or `<AppFooter />` manually inside a page component — the layout injects them.
- Assign the route to the correct layout group: `/_main/*` for `AppLayout`, `/_other/*` for `NoFooterLayout`.
- `AppLayout` already wraps content in `<Suspense fallback={<AppErrorComponent />}>`; do not re-wrap.
- `AppFooter` is injected by `AppLayout` inside the scrollable content area — do not duplicate it.

**AppLayout internals:**
```tsx
// h-dvh flex flex-col → sticky header + scrollable body
<AppHeader />          // fixed, backdrop-blur header
<div className="flex-1 overflow-y-auto scrollbar-thin scrollbar-gutter-stable">
  <Outlet />
  <AppFooter />        // rendered after page content
</div>
```

### COMPONENTS

- Only use shadcn/ui primitives already present in `src/shadcn/components/ui/`.
- Available: `accordion`, `alert-dialog`, `avatar`, `badge`, `button`, `calendar`, `card`, `carousel`, `checkbox`, `collapsible`, `command`, `context-menu`, `dialog`, `drawer`, `dropdown-menu`, `empty`, `field`, `form`, `input`, `input-group`, `input-otp`, `kbd`, `label`, `menubar`, `navigation-menu`, `popover`, `radio-group`, `scroll-area`, `select`, `separator`, `sheet`, `sidebar`, `skeleton`, `sonner`, `spinner`, `switch`, `table`, `textarea`, `toggle`, `toggle-group`, `tooltip`.
- Do **not** install new UI libraries.
- Component names: **PascalCase** (e.g. `SignUpForm`, `AppHeader`, `FormInput`).
- File names: **PascalCase.tsx** for components, **camelCase.ts** for non-component logic.
- Custom directory name exception: `custome-ui/` (note the project's intentional spelling — keep it).
- Import shadcn primitives from `@/shadcn/components/ui/<name>`, not from `@radix-ui` directly.

### DTOs & VALIDATION

- Each feature has a DTO file at `src/dto/<Name>Dto.ts`.
- Export both the Zod schema (`export const xyzSchema = z.object({...})`) and the inferred type (`export type XyzDto = z.infer<typeof xyzSchema>`).
- Use Zod v4 API (`z.string().nonempty(...)`, `z.email(...)`, `z.enum(...)`).

### SERVICES

- One service file per feature domain in `src/services/`.
- Each service exports a **custom hook** (`useXxxService`) that returns named `useMutation` / `useQuery` instances.
- HTTP calls go through `createAxiosInstance(import.meta.env.VITE_BASE_URL)`.
- Toast feedback via `sonner` (`toast.error(...)`, `toast.success(...)`) inside `onError` / `onSuccess`.

---

## Security Rules

- **Tokens in localStorage:** The current codebase stores `auth-token` and `auth-user` in `localStorage`. This is an existing pattern — follow it, but treat values as untrusted when read back (always `JSON.parse` inside a try/catch).
- **Never log tokens or credentials** to `console.log` in production paths.
- **Authorization header:** Attached by the Axios request interceptor (`createAxiosInstance`) — do not manually set `Authorization` in individual service calls.
- **401 handling:** The Axios interceptor attempts one token refresh before rejecting. Do not add duplicate 401 logic in service hooks.
- **Input validation:** All user input must pass a Zod schema before submission. Never call `mutate()` on raw, un-validated data.
- **Sanitize display:** Never render user-supplied text via `dangerouslySetInnerHTML`.
- **Search params:** Validate all URL search params with `validateSearch` on `createFileRoute` — do not read `location.search` without a schema.
- **Environment variables:** Use only `import.meta.env.VITE_*` variables. Never hardcode URLs or secrets. Never expose non-VITE_ env vars to the client bundle.
- **Mutations are side-effect-gated:** Confirm user intent (dialog, disabled states) before triggering destructive mutations.
- **CSRF:** API is accessed via Axios (Bearer token, not cookies) — standard CSRF via `SameSite` cookies is not the pattern here. Ensure the backend enforces token validation on all state-changing endpoints.

---

## Code Patterns

### Adding a new form page (e.g. Login)

**Route file** — assign to correct layout group:
```tsx
// src/routes/_other/login.tsx
import LoginPage from "@/view/pages/login/LoginPage";
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/_other/login")({
  component: () => <LoginPage />,
});
```

**Page file** — thin wrapper, no form logic:
```tsx
// src/view/pages/login/LoginPage.tsx
import LoginForm from "./forms/LoginForm";
import { useEffect } from "react";
import { useLocation } from "@tanstack/react-router";
import { scrollToView } from "@/utils/scrollToView";

function LoginPage() {
  const location = useLocation();
  useEffect(() => { scrollToView("top", "start"); }, [location.pathname]);

  return (
    <div id="top" className="min-h-screen p-6 py-12 font-inter scroll-mt-25">
      <LoginForm />
    </div>
  );
}
export default LoginPage;
```

**Form file** — all field + submission logic:
```tsx
// src/view/pages/login/forms/LoginForm.tsx
import { FormProvider, useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { loginSchema, type LoginDto } from "@/dto/LoginDto";
import { Field, FieldGroup, FieldLegend, FieldSet } from "@/shadcn/components/ui/field";
import { Button } from "@/shadcn/components/ui/button";
import FormInput from "@/custome-ui/ui/user/form/basic/custom-field-shadcn/FormInput";
import { useAuthService } from "@/services/authService";

function LoginForm() {
  const { loginMutation } = useAuthService();
  const form = useForm<LoginDto>({
    mode: "onChange",
    resolver: zodResolver(loginSchema),
    defaultValues: { email: "", password: "" },
  });

  const onSubmit = (data: LoginDto) => {
    loginMutation.mutate(data);
  };

  return (
    <FormProvider {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        <FieldGroup className="h-fit w-full max-w-lg shadow-xl mx-auto p-6 rounded-[20px] flex flex-col gap-1 border border-gray-200 bg-background">
          <FieldSet>
            <FieldLegend className="text-foreground text-center w-full">
              <span className="text-2xl lg:text-4xl">Sign In</span>
            </FieldLegend>
            <FieldGroup>
              <FormInput name="email" label="Email" type="text" placeholder="Enter your email" required />
              <FormInput name="password" label="Password" type="password" placeholder="Enter your password" required />
              <Field orientation="horizontal">
                <Button className="h-12 rounded-[10px] w-full font-normal" type="submit" disabled={loginMutation.isPending}>
                  Sign In
                </Button>
              </Field>
            </FieldGroup>
          </FieldSet>
        </FieldGroup>
      </form>
    </FormProvider>
  );
}
export default LoginForm;
```

**DTO file:**
```ts
// src/dto/LoginDto.ts
import { z } from "zod";

export const loginSchema = z.object({
  email: z.email("Please enter a valid email"),
  password: z.string().nonempty("Password is required").min(8, "Min 8 characters"),
});

export type LoginDto = z.infer<typeof loginSchema>;
```

**Service file:**
```ts
// src/services/authService.ts
import createAxiosInstance from "@/config/axios-config";
import { useMutation } from "@tanstack/react-query";
import { toast } from "sonner";

const axios = createAxiosInstance(import.meta.env.VITE_BASE_URL ?? "");

export function useAuthService() {
  const loginMutation = useMutation({
    mutationFn: async (payload: { email: string; password: string }) => {
      const { data } = await axios.post("/auth/login", payload);
      return data;
    },
    onError: (err: Error) => {
      toast.error("Login failed", { description: err.message, position: "top-right" });
    },
  });
  return { loginMutation };
}
```

---

## Checklist

Before submitting any new page or form, verify:

- [ ] Route file is in `src/routes/_main/` (with footer) or `src/routes/_other/` (without footer)
- [ ] Route uses `createFileRoute("/_main/..." | "/_other/...")` — not a manual route object
- [ ] Search params (if any) are validated with `validateSearch` in `createFileRoute`
- [ ] Page component is in `src/view/pages/<feature>/<Feature>Page.tsx`
- [ ] Form component is in `src/view/pages/<feature>/forms/<Feature>Form.tsx`
- [ ] DTO + Zod schema lives in `src/dto/<Name>Dto.ts`, not inline in the form
- [ ] `useForm` uses `mode: "onChange"` and `resolver: zodResolver(schema)`
- [ ] Form wrapped in `<FormProvider {...form}>` so field components work
- [ ] Only field components from `@/custome-ui/ui/user/form/basic/custom-field-shadcn/` are used
- [ ] Only shadcn primitives already in `src/shadcn/components/ui/` are imported
- [ ] HTTP calls go through `createAxiosInstance` — no raw `fetch` or `axios` imports
- [ ] Service is a custom hook (`useXxxService`) in `src/services/`
- [ ] Submit button has `disabled={mutation.isPending}` to prevent double-submit
- [ ] No `dangerouslySetInnerHTML` with user-supplied data
- [ ] No hardcoded URLs — use `import.meta.env.VITE_*`
- [ ] No `console.log` of tokens, passwords, or sensitive data
- [ ] `AppHeader` / `AppFooter` are **not** manually added inside the page (layouts inject them)
- [ ] Component and file names are PascalCase
- [ ] No new UI libraries introduced
