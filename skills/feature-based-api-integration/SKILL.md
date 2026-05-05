---
name: feature-based-api-integration
description: Add API integrations to LeaseDrop using its feature service hooks, Axios auth client, DTO/Zod schemas, React Query grids, upload, and error handling patterns.
---

# Feature-Based API Integration

Use this skill when adding or changing API calls in the LeaseDrop frontend. The project organizes most backend access by feature in `src/services/api`, then consumes those service functions from pages, forms, grids, and upload flows.

## API Organization

Primary service files:

- `src/services/api/auth.tsx`: login, signup, OTP, password reset/change, token refresh, signup confirmation with organization logo upload.
- `src/services/api/pdfUpload-service.tsx`: document analysis upload, anonymous upload, OTP send, clauses, summaries, share links, report blob download, quota/subscription status.
- `src/services/api/keyword-service.tsx`: clause keyword search, create, update, delete, active list.
- `src/services/api/group-service.tsx`: clause group search, create, update, delete.
- `src/services/api/banner-service.tsx`: banner search, public search, upload, update, delete, random/all banners.
- `src/services/api/payment-service.tsx`: packages, subscriptions, invoices, payment list, checkout, cancel.
- `src/services/api/customerManagement-service.tsx`: users search and customer status updates.
- `src/services/api/userProfile.tsx`: profile by email, profile update, organization logo upload/delete.
- `src/services/api/contact-us.tsx`: contact and waitlist notification requests.

The dominant pattern is a feature hook that returns named API functions:

```ts
export function usePaymentService(): PaymentImpl {
  function getPaymentList(page: number, size: number, sortBy = 'status', sortDirection = 'desc') {
    return axiosInstance.post(`${baseUrl}/subscriptions/search`, { ... });
  }

  return { getPaymentList, getInvoice, getSubscription, cancelSubscription };
}
```

## Axios and Authentication

Use `src/config/axios-config.tsx` for authenticated Axios calls. It:

- Creates an Axios instance with a supplied `baseURL`.
- Reads `auth` from `localStorage`.
- Adds `Authorization: Bearer <accessToken>` to requests.
- Handles `401` once per request with `_retry`.
- Calls `${baseUrl}/auth/refresh-token` with the saved refresh token.
- Updates `localStorage.auth` with the new access token.
- Removes `auth` if refresh fails.

Preferred new service pattern:

```ts
const baseUrl: string = import.meta.env.VITE_BASE_URI || '';
const axiosInstance = createAxiosInstance(baseUrl);

export function useFeatureService() {
  const featureBaseUrl = '/feature';
  return {
    getList: (searchBody: SearchBody) => axiosInstance.post(`${featureBaseUrl}/search`, searchBody),
  };
}
```

Avoid manually rebuilding auth headers in new code unless you must use `fetch` or a separate Axios instance. Existing services like `keyword-service.tsx`, `group-service.tsx`, and `SavedFileGrid` do manual headers; treat that as legacy/inconsistent.

## Request and Response Types

Shared DTOs and validation schemas live in `src/dto`.

Examples:

- `KeywordDto` plus `keywordValidation()`.
- `GroupDto` plus `groupValidation()`.
- `BannerDto` plus `bannerValidation()`.
- `UserDto` plus `userValidation()`.
- `FileUploadDto` plus `fileUploadSchema`.

Service files often type responses as `Promise<AxiosResponse>`. New APIs should improve request typing where practical:

```ts
type SearchBody = {
  filter: Record<string, any>;
  sort: { field?: string; direction?: 'asc' | 'desc' | null };
  page: { pageNumber: number; size: number };
};
```

Do not invent DTOs inside components when the type is shared by services, grids, forms, or modals. Put those in `src/dto`.

## Search, Sort, Filter, and Pagination

Grid-backed APIs use a consistent request body shape:

```ts
const searchBody = {
  filter: { ...filters },
  sort: { ...sort },
  page: {
    pageNumber: pageIndex + 1,
    size: pageSize,
  },
};
```

Important details:

- UI `pageIndex` is zero-based.
- Backend `pageNumber` is one-based.
- `pageCount` is commonly calculated as `Math.ceil(response.data?.totalElements / pageSize)`.
- Filters are stored as `Record<string, any>` keyed by DataGrid column IDs.
- Empty filter values remove the filter key.
- Sort state usually maps DataGrid callbacks to `{ field: columnId, direction }`.

Examples:

- `keyword-list-grid.tsx` uses `queryKey: ["keywords", pageIndex, pageSize, JSON.stringify(debouncedFilters), sort]`.
- `group-list-grid.tsx` uses the same pattern for groups.
- `banner-grid.tsx` builds a search body and passes it to `getBannerList`.
- `payment-grid.tsx` calls `getPaymentList(pageIndex + 1, pageSize, sort.column, sort.direction || undefined)`.

## React Query Usage

React Query is provided by `src/Provider/query-client-provider.tsx`. Existing list screens use `useQuery` with:

- A feature-specific `queryKey`.
- A `queryFn` that calls the feature service.
- `staleTime: 1000`.
- Local component state for rows (`setUserData`, `setGroupData`, etc.).
- `refetch()` after create/update/delete.

Example pattern:

```ts
const { data, isLoading, refetch } = useQuery({
  queryKey: ['groups', pageIndex, pageSize, JSON.stringify(debouncedFilters), sort],
  queryFn: async () => {
    const response = await getGroupList(searchBody);
    const content = response.data?.content || [];
    setGroupData(content);
    return { data: content, pageCount: Math.ceil(response.data?.totalElements / pageSize) };
  },
  staleTime: 1000,
});
```

Mutations are usually handled manually with async functions and `refetch()`, not React Query `useMutation`. Match the nearby feature unless you are deliberately refactoring a feature end to end.

## Loading, Success, Error, and Empty States

Existing conventions:

- `DataGrid` receives `loading={isLoading}` or local `loading`.
- Empty states are passed with `emptyState={<div>...</div>}`.
- Form submissions use `loading` state and disable buttons when processing.
- User messages use `const { showModal } = useModal()` and call `showModal(message, type)`.
- Some older code uses `react-toastify` helpers in `src/components/Toast/toastUtils.tsx`.

Error handling examples:

- `GroupGrid` catches delete conflicts with `AxiosError` and checks status `409`.
- `change-password-form.tsx` catches errors and shows `error.message`.
- `SavedFileGrid` catches `fetch` errors and displays generic failure messages.
- `auth.tsx` sometimes returns `error.response` instead of throwing; account for that behavior when consuming auth functions.

New code should show generic user-safe messages and keep detailed diagnostics out of production logs.

## Mutations

Existing mutation names are plain verbs:

- Create: `addKeyword`, `addGroup`, `uploadBanner`, `signUp`, `confirmSignUp`.
- Update: `updateKeyword`, `updateGroup`, `updateBanner`, `updateUserProfile`, `UpdateSummary`.
- Delete: `deleteKeyword`, `deleteGroup`, `deleteBanner`, `deleteOrganizationLogo`.
- Submit/send: `sendContactUsEmail`, `sendNotificationRequest`, `OptSend`.
- Approve/reject are not represented in the inspected services; if added, use clear verb names such as `approveX` and `rejectX`.

After mutation:

- Show a modal for success or failure.
- Close edit/delete modals.
- Reset selected entity state.
- Call `refetch()` for affected lists.
- Reset forms if the operation completes successfully.

## File Upload and Multipart Requests

Multipart patterns:

- `pdfUpload-service.tsx` creates `FormData`, appends `llm`, `document`, and `paidUser`, sets `Content-Type: multipart/form-data`, and supports `onUploadProgress`.
- `auth.tsx` `confirmSignUp(data, orgLogo)` appends `request` as `JSON.stringify(data)` and optionally appends `orgLogo`.
- `banner-service.tsx` `uploadBanner(formData)` posts `FormData`.
- `userProfile.tsx` `uploadOrganizationLogo(formData)` posts `FormData`.

Upload validation examples:

- `file-upload-minimal.tsx` accepts one PDF or image depending on `acceptType`, rejects invalid MIME types, limits images to 1 MB, reads PDFs with `pdfjs-dist`, and returns `{ sizeMB, pageCount }`.
- `FileUploadDto.tsx` has a Zod schema limiting documents to 10 MB and extensions `pdf`, `doc`, `docx`, `png`, `jpg`, and `jpeg`.
- `LandingPageScreen.tsx` prevents analysis for files over 55 MB or PDFs over 300 pages.

If adding file upload:

- Validate file type and size in UI.
- Use `FormData`.
- Include `onUploadProgress` if the UI needs progress.
- Do not log file contents.
- Use the existing modal system for rejection messages.

## WebSocket/Progress Integration

Document processing progress is handled in `LandingPageScreen.tsx` with `@stomp/rx-stomp`:

- `brokerURL` is `${VITE_WEBSOCKET_URI}/leasedrop-session`.
- The anonymous or logged-in user ID comes from `getOrCreateUserId()`.
- The app publishes to `PUB_APP_USERS = '/app/user'`.
- It watches `/user/${userId}/topic/private-message`.
- It parses `message.body`, then parses `newMessage.comment` as JSON.
- Progress is matched to `localStorage.last_upload_hash`.
- `FINISH` navigates to `/document/analysis`; `FAILED` moves to the upload failed UI.

Do not add a second WebSocket client for the same flow. Reuse or extract the existing STOMP pattern if another page needs processing status.

## Checklist For Adding A New API Feature

- Create or identify `src/services/api/<feature>-service.tsx`.
- Read `VITE_BASE_URI` at module scope and create `axiosInstance` with `createAxiosInstance(baseUrl)` for authenticated APIs.
- Define DTOs and Zod validation in `src/dto/<Feature>Dto.tsx` if data is shared.
- Add service functions with names matching existing verbs.
- Use relative endpoint paths with `axiosInstance` where possible.
- For list/search APIs, use `{ filter, sort, page: { pageNumber, size } }`.
- In the page/grid, use `useQuery` with a feature-specific `queryKey`.
- Pass data, loading, pagination, sort, filter, and empty state into `DataGrid`.
- On create/update/delete, show a modal, close modals/forms, reset selected state, and `refetch()`.
- For uploads, validate type/size before sending `FormData`.
- For errors, avoid exposing raw backend objects to users.

## Don't

- Do not hardcode backend hosts like `http://10.x.x.x` in new service code.
- Do not duplicate access-token parsing in every new service when `createAxiosInstance` already attaches tokens.
- Do not mix `fetch` and Axios in a new feature unless there is a clear reason such as blob downloads.
- Do not put API calls directly inside reusable components.
- Do not create request schemas only in a page when they are shared by a service and a form.
- Do not assume all services throw errors; auth helpers sometimes return Axios error responses.

## Unclear Areas

- Some services manually attach headers while others use `createAxiosInstance`; prefer the central Axios instance for new code.
- `VITE_BASE_URI2` exists but its active purpose is not clear from the inspected code.
- `src/services/api/common.tsx`, files with `copy`, and files with `BK`/dated suffixes appear to be legacy or backups.
- React Query is installed and provided, but mutations are mostly manual instead of `useMutation`.
