---
name: ui-migration
description: >
  Use this skill whenever the user wants to copy, port, or replicate the visual design and UI of
  a source/demo codebase into a target codebase — WITHOUT changing the target's underlying logic,
  API services, or configuration. Triggers include: "make the UI look exactly like the demo",
  "copy the design from X into Y", "migrate only the frontend look", "replicate the layout/styles
  from the source", "keep the logic but change the look", "apply the design system from demo to
  our app", or any task where UI appearance must match a reference while the target's own wiring
  (routes, API calls, auth, env vars) must stay intact. Always read this skill before writing a
  single line of code.
---

# UI Migration Skill

> **Scope:** Visual design, layout, and component styling only.
> **Never touch:** API service wiring, env config, auth logic, business rules, or data-fetching hooks.

Stack assumed: **React · TanStack (Query / Router / Form) · shadcn/ui · Tailwind CSS**
Adapt if the scanned codebase differs.

---

## Mental Model

```
SOURCE (demo)   →  read-only. Extract UI patterns, tokens, component shapes.
TARGET (output) →  UI layer is fully replaceable. Logic layer is untouchable.
```

### The Two Layers — Know Them Apart

| Layer | What lives here | Migration action |
|---|---|---|
| **UI layer** | JSX markup, Tailwind classes, shadcn/ui primitives, layout wrappers, colour tokens, spacing, typography, icons, animations | **Fully replace to match SOURCE** |
| **Logic layer** | `useQuery`, `useMutation`, API client calls, env vars, auth guards, form schemas (Zod), router config, middleware | **Do not touch — preserve exactly** |

When in doubt: *if removing it would break a network call or auth check, it belongs to the logic layer.*

---

## Workflow — 4 Gated Steps

Steps are **sequential and gated**. Do not advance until each step is complete.

---

### STEP 1 — Analyse SOURCE (read-only, UI focus)

Scan SOURCE. Extract only what is needed to replicate the look. **Do not write any code yet.**

```
UI Extraction Checklist:
1.  Directory tree (2 levels deep) — note where UI components live
2.  Design tokens → colours, spacing scale, border-radius, shadows
    - tailwind.config file → extend.colors, extend.spacing, extend.borderRadius
    - CSS variables → :root block in global.css / index.css
3.  Typography → font families (Google Fonts? local?), weights, size scale
4.  Layout structure → outer shell, sidebar, topbar, footer — exact component names
5.  Page templates → how pages are composed (slots, children, named outlets)
6.  shadcn/ui components used → grep "from '@/components/ui/"
7.  Icons → which icon library (lucide-react, heroicons, etc.) and which icons
8.  Animations / transitions → Tailwind animate-*, framer-motion usage, CSS keyframes
9.  Responsive breakpoints → custom or Tailwind defaults?
10. Dark mode → class-based? system? toggle logic location?
```

Print a **SOURCE UI Summary** block before proceeding:

```
## SOURCE UI Summary
Design tokens:   <tailwind.config extend keys OR CSS vars listed>
Font:            <family, weights, import method>
Layout shell:    <component name + path>
Sidebar:         <yes/no — component name>
Topbar:          <yes/no — component name>
Footer:          <injected in layout | per-page | none>
shadcn/ui:       <exhaustive list of components found>
Icon library:    <name + version>
Animations:      <Tailwind animate-* | framer-motion | none>
Dark mode:       <class | system | none>
Responsive:      <Tailwind defaults | custom breakpoints listed>
```

---

### STEP 2 — Analyse TARGET (logic preservation map)

Scan TARGET. Build a **Logic Preservation Map** — a complete list of everything that must not change.

```
Logic Preservation Checklist:
1. API client setup → baseURL, interceptors, auth headers
2. All useQuery / useMutation hooks → their queryKeys and fetcher functions
3. Auth flow → tokens, session, guards, redirect logic
4. Environment variable usage → every process.env / import.meta.env reference
5. Form schemas → every Zod schema, validation rule, error message
6. Router config → all routes, loaders, guards, redirects
7. Middleware → rate limits, CSP, size checks
8. Third-party SDK calls → Stripe, Sentry, analytics, etc.
```

Print a **TARGET Logic Map** block:

```
## TARGET Logic Map (DO NOT MODIFY)
API client:      <file path>
Auth guard:      <file path>
Env vars:        <list every key used>
Query hooks:     <list files containing useQuery / useMutation>
Zod schemas:     <list schema files>
Router:          <config file path>
Middleware:      <file path>
Third-party:     <list SDKs + their init files>
```

---

### STEP 3 — Apply UI Layer to TARGET

Rewrite the TARGET's UI layer to match SOURCE exactly. Preserve every file in the Logic Map.

#### 3a — Design Token Sync

```
1. Copy SOURCE tailwind.config extend block into TARGET tailwind.config.
   - Merge, do not replace — TARGET may have functional tokens SOURCE doesn't.
2. Copy SOURCE :root CSS variables into TARGET global CSS file.
   - If TARGET has existing vars not in SOURCE, keep them (they may be used by logic).
3. Copy font import (Google Fonts link / @font-face) into TARGET index.html or layout.
```

#### 3b — Layout Shell Replacement

```
Replace TARGET's layout shell component(s) with SOURCE's:
- Copy JSX structure exactly (sidebar, topbar, footer slots).
- Keep TARGET's <Outlet /> / {children} slot — it must stay for router to inject pages.
- Keep any auth guard wrappers that wrap the shell in TARGET (do not remove them).
- Do NOT replace router config to point at new shell — update the existing reference.
```

#### 3c — Page-Level UI Rebuild

For each page in TARGET:

```
1. Identify the matching page in SOURCE (by route / purpose).
2. Copy the JSX markup and Tailwind classes from SOURCE page exactly.
3. Rewire data: replace SOURCE's data vars with TARGET's existing hook return values.
   - SOURCE: const { data } = useProductsQuery()   (SOURCE hook)
   - TARGET: const { data } = useProducts()         (TARGET hook — unchanged)
4. Keep every onClick, onSubmit, and onChange handler from TARGET.
5. Keep every form submission call from TARGET.
```

**Never rename a hook. Never change a queryKey. Never change a fetcher URL.**

#### 3d — Component-Level UI Rebuild

```
shadcn/ui primitives:
- Add any SOURCE shadcn/ui components missing from TARGET:
    npx shadcn@latest add <component>
- Do NOT remove TARGET's existing shadcn/ui components even if SOURCE doesn't use them.

Custom components (non-shadcn):
- If SOURCE has a custom component not in TARGET (e.g. <StatCard>, <EmptyState>):
    Copy it into TARGET /components/ui/ or /components/shared/
    Strip any SOURCE-specific data fetching inside it
    Accept data via props only

Icons:
- If SOURCE uses a different icon library than TARGET:
    Install SOURCE's library
    Do NOT remove TARGET's — both can coexist
    Replace icon JSX in UI components only
```

#### 3e — Animation & Dark Mode

```
Animations:
- Copy Tailwind animate-* classes as-is.
- If SOURCE uses framer-motion: install it, copy motion.* wrappers around UI elements only.

Dark mode:
- If SOURCE uses class-based dark mode and TARGET does not:
    Add darkMode: 'class' to TARGET tailwind.config
    Add the toggle logic as a standalone hook/component — do not alter any auth logic
- If TARGET already has dark mode, keep its toggle mechanism; only update the CSS vars.
```

#### Implementation Order

Follow this order to avoid broken layouts mid-migration:

```
1. tailwind.config + global.css  → tokens first, everything inherits from these
2. Font import                   → index.html or root layout
3. Layout shell                  → AppLayout, Sidebar, Topbar, Footer
4. Shared / atomic components    → copy missing shadcn/ui + custom components
5. Page components               → one section at a time
6. Animations + dark mode        → last, non-blocking
```

Print a one-line status after each section:
```
✅ tokens — tailwind.config + :root CSS vars synced
✅ layout shell — AppLayout, Sidebar, Topbar, Footer updated
✅ shared components — 3 shadcn/ui added, StatCard copied
✅ pages — Dashboard, Users, Settings rebuilt
✅ animations + dark mode — applied
```

---

### STEP 4 — Verify

Run all checks after writing files.

#### UI Fidelity Checks

```bash
# 1. Confirm SOURCE design tokens are present in TARGET config
grep -n "colors\|spacing\|borderRadius\|fontFamily" <TARGET>/tailwind.config.*

# 2. Confirm SOURCE shadcn/ui components are installed
ls <TARGET>/components/ui/

# 3. Confirm font is imported
grep -rn "font-family\|fonts.googleapis\|@font-face" <TARGET>/index.html \
  <TARGET>/src/index.css <TARGET>/app/globals.css 2>/dev/null | head -20
```

#### Logic Integrity Checks

```bash
# 4. API client unchanged — baseURL still present
grep -rn "baseURL\|axios.create\|createClient\|fetch(" \
  <TARGET>/src/lib/ <TARGET>/src/services/ 2>/dev/null

# 5. Env vars still referenced (not removed)
grep -rn "process\.env\|import\.meta\.env" <TARGET>/src --include="*.ts" --include="*.tsx"

# 6. Auth guard still in place
grep -rn "auth()\|getServerSession\|verifyToken\|requireAuth\|ProtectedRoute\|useAuth" \
  <TARGET>/src --include="*.tsx" --include="*.ts"

# 7. No new hardcoded secrets introduced during UI work
grep -rn -E "(api_key|secret|password|token)\s*[:=]\s*['\"][^'\"]{8,}" \
  <TARGET>/src --include="*.ts" --include="*.tsx" | grep -v node_modules

# 8. No dangerouslySetInnerHTML added
grep -rn "dangerouslySetInnerHTML" <TARGET>/src --include="*.tsx"
```

Report results:

```
## Verification Report

UI Fidelity:
[PASS] Design tokens synced (tailwind.config + CSS vars)
[PASS] All SOURCE shadcn/ui components installed in TARGET
[WARN] Font import not found in index.html — add manually if SSR project

Logic Integrity:
[PASS] API client baseURL intact
[PASS] All env var references preserved
[PASS] Auth guard present on protected routes
[PASS] No hardcoded secrets introduced
[PASS] No unguarded dangerouslySetInnerHTML added
```

Fix any `[FAIL]` items before finishing. Flag `[WARN]` items to the user.

---

### Final Output

End with a file tree showing only files **changed** and **added**. Mark each:

```
TARGET/ (changed files only)
├── tailwind.config.ts            [UPDATED — tokens merged]
├── src/
│   ├── index.css                 [UPDATED — :root CSS vars + font import]
│   ├── components/
│   │   ├── ui/
│   │   │   ├── badge.tsx         [ADDED — from shadcn]
│   │   │   └── StatCard.tsx      [ADDED — custom component from SOURCE]
│   │   └── layout/
│   │       ├── AppLayout.tsx     [UPDATED — UI shell rebuilt]
│   │       ├── Sidebar.tsx       [UPDATED — UI rebuilt]
│   │       └── Topbar.tsx        [UPDATED — UI rebuilt]
│   └── pages/
│       ├── Dashboard.tsx         [UPDATED — UI rebuilt, hooks preserved]
│       └── Settings.tsx          [UPDATED — UI rebuilt, form schema untouched]
│
## NOT TOUCHED (logic layer — preserved exactly):
# src/lib/api.ts
# src/lib/env.ts
# src/hooks/useProducts.ts
# src/hooks/useAuth.ts
# src/lib/schemas/
# middleware.ts
# .env / .env.example
```

---

## Quick Reference — Decision Tree

```
User gives SOURCE + TARGET paths →
  ├── Extract UI tokens + patterns from SOURCE (Step 1)
  ├── Map logic layer in TARGET — things never to touch (Step 2)
  ├── Apply UI layer in order: tokens → shell → components → pages (Step 3)
  └── Verify UI fidelity + logic integrity (Step 4)
```

```
Should I copy this file from SOURCE? →
  ├── Is it a layout/page/component with only JSX + classes? → YES, copy UI
  ├── Does it contain useQuery / fetch / axios / env var? → NO, rewrite UI around TARGET's version
  └── Does it contain Zod schema / auth / router config? → NO, never touch
```

```
SOURCE has a component TARGET doesn't →
  ├── Pure UI (no fetching)? → Copy into TARGET /components/ui/ or /components/shared/
  └── Has data fetching? → Strip fetching, convert to props, then copy UI shell only
```

```
SOURCE uses a different shadcn/ui component than TARGET's page →
  ├── Install the component: npx shadcn@latest add <component>
  └── Replace JSX markup only — keep TARGET's event handlers and data vars
```

---

## Files This Skill Touches

| File | Action | Notes |
|---|---|---|
| `tailwind.config.*` | Merge | Add SOURCE tokens; keep TARGET-only tokens |
| `src/index.css` / `app/globals.css` | Merge | Add SOURCE :root vars; keep TARGET vars |
| `index.html` | Update | Font import only |
| `layouts/AppLayout.tsx` | Replace UI | Keep `{children}` / `<Outlet />` slot |
| `layouts/Sidebar.tsx` | Replace UI | Keep auth wrapper if present |
| `layouts/Topbar.tsx` | Replace UI | Keep logout/user handlers |
| `components/ui/*` | Add missing | Never remove existing |
| `pages/*.tsx` | Replace UI markup | Rewire to TARGET hooks — never replace hooks |
| `components/shared/*` | Add from SOURCE | Strip fetching; props-only |

## Files This Skill Never Touches

| File | Reason |
|---|---|
| `lib/api.ts` / `services/*` | API wiring |
| `lib/env.ts` | Env var validation |
| `lib/schemas/*.ts` | Zod form schemas |
| `hooks/use*.ts` | Data-fetching hooks |
| `middleware.ts` | Auth / rate limit / CSP |
| `router.ts` / route config | Navigation logic |
| `.env` / `.env.*` | Secrets and config |
| `*.test.ts` / `*.spec.ts` | Test coverage |
