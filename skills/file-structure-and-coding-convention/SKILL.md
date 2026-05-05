---
name: file-structure-and-coding-convention
description: Follow the LeaseDrop frontend folder structure, naming style, and safe editing conventions when modifying this project or creating a similar React/Vite project.
---

# File Structure and Coding Convention

Use this skill when adding or changing code in the LeaseDrop frontend. This repository is a Vite + React + TypeScript app with React Router, Context providers, Redux Persist, React Query, React Hook Form, Zod, Tailwind CSS, Axios, and a custom data-grid system.

## Project Structure

Top-level application code lives in `src/`. Static assets live in `public/`. Build, mode, and deploy configuration is at the repository root.

Important folders:

- `src/main.tsx`: app bootstrap. It wires `RouterProvider`, Redux `Provider`, `PersistGate`, `ModalProvider`, React Query `Providers`, `AuthProvider`, `SkeletonContextProvider`, `UserProvider`, and `ViewProvider`.
- `src/routes.tsx` and `src/routes/*.tsx`: React Router route object composition. Feature route files include `MainRoutes.tsx`, `LoginRoutes.tsx`, `ProfileRoutes.tsx`, `DocAnalysisRoutes.tsx`, and `InfoRoutes.tsx`.
- `src/views/layouts`: page layout wrappers such as `AppLayout`, `AuthLayout`, `ProfileLayout`, and `AnalysisLayout`.
- `src/views/pages`: route-level screens grouped by domain, for example `auth`, `Profile`, `home_page`, `Pricing`, `Policy`, `Product`, and `contact_us`.
- `src/components`: reusable UI building blocks. Examples include `buttons/ButtonRC.tsx`, `input-field/input-field.tsx`, `data-grid/data-grid.tsx`, `modal/DeleteModal.tsx`, `upload-field/file-upload-minimal.tsx`, and `date-picker/date-picker.tsx`.
- `src/services/api`: feature-oriented API service hooks and functions such as `auth.tsx`, `pdfUpload-service.tsx`, `keyword-service.tsx`, `group-service.tsx`, `banner-service.tsx`, `payment-service.tsx`, and `userProfile.tsx`.
- `src/config`: shared runtime clients. `axios-config.tsx` creates an Axios instance with auth and refresh-token interceptors.
- `src/dto`: TypeScript DTOs and related Zod validation schemas, such as `KeywordDto.tsx`, `GroupDto.tsx`, `BannerDto.tsx`, `UserDto.tsx`, and `FileUploadDto.tsx`.
- `src/Provider`: React context providers. Existing names use PascalCase for many files, for example `AuthContext.tsx`, `UserContextProvider.tsx`, and `ViewContext.tsx`.
- `src/redux`: Redux Toolkit store and slices. `store.tsx` configures Redux Persist and persists the `user` slice.
- `src/hooks`: reusable hooks such as `use-debounce.tsx`, `usePdfCache.tsx`, and data-grid hooks.
- `src/utils` and `src/views/utils`: general helpers. Existing code uses both locations, so prefer `src/utils` for app-wide utilities and only add to `src/views/utils` when the helper is tightly coupled to views.
- `src/assets`: icon and SVG service code.
- `public/assets`: images, SVGs, audio, and other static files referenced by the frontend.

## Naming Conventions

Follow the style already used near the code you modify.

- Route modules: PascalCase plus `Routes`, for example `ProfileRoutes.tsx`.
- Layouts: PascalCase plus `Layout`, for example `AuthLayout.tsx`.
- Page components: PascalCase route/page names, often with `Page` or `View`, for example `SignInPage.tsx`, `SignInView.tsx`, `PricingPage.tsx`, and `AnalysisPage.tsx`.
- Feature services: mostly `feature-service.tsx` and exported `useFeatureService`, for example `useKeywordService` from `keyword-service.tsx`, `useGroupService` from `group-service.tsx`, and `usePdfUploadService` from `pdfUpload-service.tsx`.
- DTOs: PascalCase with `Dto`, for example `KeywordDto`, `GroupDto`, `BannerDto`, and `UserDto`.
- Validation helpers: lower camel case plus `Validation`, for example `keywordValidation()`, `groupValidation()`, `bannerValidation()`, and `userValidation()`.
- Hooks: `use-*.tsx` for some generic hooks (`use-debounce.tsx`) and `useSomething.tsx` for others (`usePdfCache.tsx`). Keep the local folder's existing style.
- Shared UI components: mixed historical conventions exist. New reusable components should use a clear PascalCase component name and a folder that describes the component family, matching examples such as `ButtonRC`, `InputField`, `DataGrid`, and `DeleteModal`.
- Constants: uppercase names are used for route/WebSocket constants such as `PRV_PREFIX`, `SUB_USER_MESSAGE_PRV_URL`, and `PUB_APP_USERS`.

## Where To Add New Code

- New route screen: add the route component under `src/views/pages/<feature>/`, then register it in the relevant `src/routes/*Routes.tsx` file.
- New route layout: add to `src/views/layouts` and use it from a route module.
- New reusable UI: add under `src/components/<component-family>/`. Reuse `cn` from `src/utils/cn.tsx` for class merging.
- New API integration: add or extend `src/services/api/<feature>-service.tsx`. Prefer the feature service hook style and central `createAxiosInstance`.
- New request/response model or validation schema: add to `src/dto/<Feature>Dto.tsx` when shared across pages/services.
- New app-wide hook: add to `src/hooks`.
- New view-only helper: add near the view or in `src/views/utils` if it is already coupled to page behavior.
- New global context: add to `src/Provider` and wire it in `src/main.tsx` only if many unrelated branches need it.
- New Redux state: add a slice under `src/redux/slices`, register it in `src/redux/store.tsx`, and decide explicitly whether it belongs in the Redux Persist whitelist.

## Common Project Patterns

- Providers are composed in `src/main.tsx`, not inside individual pages.
- Routing is declarative with React Router route objects, not scattered `<Routes>` declarations.
- Authenticated routes use `CustomerPrivateRoute.tsx`/`PrivateRoute.tsx`, which reads `useAuth()` and redirects unauthenticated users through `useNavigate`.
- API services typically expose a hook returning functions, for example:

```ts
export function useGroupService() {
  const getGroupList = async (searchBody: any) => axios.post(`${GROUP_BASE_URL}/search`, searchBody, { headers });
  return { getGroupList, addGroup, updateGroup, deleteGroup };
}
```

- Table screens use `DataGrid` with `columns`, `pagination`, `onSort`, `onFilter`, `keyField`, and `emptyState`.
- Search/list APIs usually send a body shaped like:

```ts
{
  filter: { ...filters },
  sort: { field: columnId, direction },
  page: { pageNumber: pageIndex + 1, size: pageSize }
}
```

- Forms commonly use React Hook Form. More complex forms use Zod schemas from `src/dto`, as seen in `clause-keyword-form.tsx`, `clause-group-form.tsx`, `personal-information-form.tsx`, and `change-password-form.tsx`.
- User feedback is usually shown through `useModal().showModal(message, 'success' | 'error' | 'warning')`.
- Styling is mostly Tailwind classes, often merged with `cn(...)`.

## Import and Export Conventions

- Prefer named exports for services, DTOs, validation helpers, hooks, and reusable utilities.
- Component files often default export the main component. Match neighboring files in the same folder.
- Imports are mostly relative paths. Some files include explicit `.tsx` extensions because `allowImportingTsExtensions` is enabled. Follow nearby imports rather than changing large import blocks.
- Use `cn` from `src/utils/cn.tsx` when composing conditional Tailwind classes.
- Do not introduce path alias imports unless the project is first configured for them in TypeScript and Vite.

## Safe Modification Rules

- Read the nearby implementation before editing. There are backup/legacy files such as `copy`, `BK`, `Original`, and dated files; do not treat those as the canonical pattern unless the active import uses them.
- Do not extend duplicate folders like `src/components/data-grid copy`, `data-grid copy 2`, or `data-grid copy 3`. Prefer the active `src/components/data-grid` folder unless imports prove otherwise.
- Avoid adding new hardcoded backend URLs. Some older files have hardcoded values, such as `src/services/api/common.tsx`; treat those as legacy to replace, not a pattern to copy.
- Do not move routes, providers, or Redux slices casually. These are connected through `src/main.tsx`, `src/routes.tsx`, and `src/redux/store.tsx`.
- Keep changes small and feature-scoped. If adding a feature, add the page, service, DTO/schema, and reusable component pieces in their existing domains.
- Do not log auth tokens, uploaded document contents, raw API responses, or personally identifiable data. Several existing files log responses; new work should improve this.
- Avoid `dangerouslySetInnerHTML` for new UI unless rendering one of the existing trusted icon strings from `src/assets/svg-service.tsx`.

## Do

- Put feature API code in `src/services/api`.
- Put shared validation in `src/dto` beside the DTO it validates.
- Reuse `DataGrid`, `InputField`, `DatePickerField`, `DeleteModal`, `DecisionModal`, `ActionButtons`, and `useModal` where they fit.
- Use `createAxiosInstance(baseUrl)` for authenticated Axios calls.
- Keep route registration inside the route module that owns the route area.
- Use `pageIndex + 1` when building backend pagination request bodies because UI state is zero-based and backend requests are one-based in existing grids.

## Don't

- Do not create new root-level app folders unless the existing `src` structure cannot support the feature.
- Do not add new `copy`, `BK`, or dated backup files.
- Do not bypass providers by duplicating global state in unrelated components.
- Do not add secrets to `package.json`, `.env.*`, or source files.
- Do not introduce a second notification system when `ModalProvider` or existing toast utilities are sufficient.
- Do not assume `README.md` is authoritative; it is still the default GitLab template. Prefer code evidence.

## Checklist For Future Agents

- Identify the route/page/service/DTO/component area that already owns the behavior.
- Check active imports to avoid editing backup files.
- Reuse existing providers, Redux slices, services, and components before adding new ones.
- Put new API calls in a feature service and new shared schemas in `src/dto`.
- Follow local file naming in the target folder.
- Keep environment-dependent values in `import.meta.env` configuration.
- Show UI success/error through `showModal` or the existing toast utilities.
- Run `npm run build` for TypeScript/Vite verification when feasible.
