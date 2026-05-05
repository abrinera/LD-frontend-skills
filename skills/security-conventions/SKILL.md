---
name: security-conventions
description: Preserve LeaseDrop frontend authentication, authorization, token, validation, upload, logging, and secure API conventions when changing the project.
---

# Security Conventions

Use this skill when changing authentication, authorization, API calls, file upload, validation, logging, configuration, or user-data flows in the LeaseDrop frontend.

## Authentication

The frontend uses access and refresh tokens returned from backend auth APIs.

Main files:

- `src/services/api/auth.tsx`: login, signup, OTP, password reset/change, token refresh.
- `src/Provider/AuthContext.tsx`: stores auth state, syncs it to `localStorage`, schedules token refresh, and clears auth data on logout.
- `src/config/axios-config.tsx`: attaches `Authorization: Bearer <accessToken>` to Axios requests and refreshes tokens on `401`.
- `src/views/pages/auth/SignInPage/SignInView.tsx`: logs in, dispatches user info, stores auth and `userId`, and calls `setAuth`.

Current auth storage keys include:

- `auth`: JSON with `accessToken`, `refreshToken`, and `expiresIn`.
- `email`.
- `userId`.
- `role`.
- `userInfo`.
- `last_upload_hash`.

Because tokens are stored in `localStorage`, treat this as sensitive. Do not add token logging, token copying, or new long-lived secrets to browser storage.

## Authorization and Role Handling

The project has lightweight frontend authorization:

- `PrivateRoute.tsx` and `CustomerPrivateRoute.tsx` gate protected routes based on `useAuth().auth`.
- `src/redux/slices/userSlice.tsx` derives `isAdmin` from `user?.role?.[0] === 'ADMIN'`.
- Paid access is tracked with `isPaid`, where examples compare to `'Paid'` or allow admins.
- `SavedFileGrid` disables some actions for non-paid users by passing `disabledButtons={!isPaidUser ? ['Page'] : []}`.
- `ProtectedComponent.tsx` disables context menu and certain developer shortcuts for non-paid users and blurs on tab visibility changes.

Important boundary: frontend route guards and disabled buttons are convenience controls only. Backend APIs must enforce all real authorization. Do not rely on frontend `isAdmin`, `isPaid`, disabled buttons, or `ProtectedComponent` as the only protection for sensitive data.

## Secure API Requests

Preferred authenticated request pattern:

```ts
const baseUrl: string = import.meta.env.VITE_BASE_URI || '';
const axiosInstance = createAxiosInstance(baseUrl);
return axiosInstance.post('/feature/search', searchBody);
```

`createAxiosInstance` handles:

- Reading the latest token from `localStorage.auth`.
- Adding the bearer token dynamically.
- Refreshing once after `401`.
- Clearing `auth` after refresh failure.

Avoid adding new manual token header code unless using `fetch` for blob downloads or another case Axios cannot handle cleanly.

When using `fetch`, include only the necessary headers:

```ts
headers: {
  Authorization: `Bearer ${accessToken}`,
}
```

Do not send auth tokens in query strings.

## Secrets and Configuration

Security-sensitive findings from the current code:

- `package.json` contains a hardcoded Sonar token in the `verify` script. Do not copy this pattern; move such tokens to CI secrets.
- Some `.env.*` files are present in the checkout. Treat any real hostnames, keys, or tokens in env files as sensitive and confirm team policy before committing changes.
- `src/services/api/common.tsx` hardcodes an internal API host. Do not add new hardcoded backend hosts.
- `VITE_*` variables are public in browser bundles. Never put private API keys, credentials, or service tokens in `VITE_*`.

Rules:

- Public API base URLs can use `VITE_BASE_URI`.
- Secrets must live on the backend or in CI/CD secret stores.
- CI credentials should follow the Jenkins credential pattern, such as `GITLAB_ID = 'gitlab-credentials'`, rather than literal tokens.

## Input Validation

The project uses React Hook Form and Zod for many forms and editable grids.

Validation examples:

- `KeywordDto.tsx`: validates keyword name length, prompt length, relation arrays, clause group shape, non-negative position, and enabled state.
- `GroupDto.tsx`: validates group name, description, non-negative position, and enabled state.
- `BannerDto.tsx`: validates image-like file names, duration, size, and redirect URL format with `new URL(...)`.
- `UserDto.tsx`: validates first and last names, company name, and organization logo type.
- `FileUploadDto.tsx`: validates paid user value, file size, and file extension.
- `change-password-form.tsx`: requires current password, new password length, uppercase, lowercase, number, and matching confirmation.
- `SignInView.tsx` and `FormRC.tsx`: validate email and password required fields with React Hook Form.

When adding input:

- Put shared Zod schemas in `src/dto`.
- Use `zodResolver` in forms that have a DTO/schema.
- Keep form-level and field-level errors user-safe and specific.
- Validate both UI and backend. Frontend validation is for UX, not security.

## File Upload Security

Existing upload controls perform client-side restrictions:

- `file-upload-minimal.tsx` accepts one file, restricts MIME type to PDF or images depending on `acceptType`, rejects unsupported types, limits images to 1 MB, reads PDF page count with `pdfjs-dist`, and reports errors through `showModal`.
- `LandingPageScreen.tsx` blocks document processing when size exceeds 55 MB or page count exceeds 300.
- `pdfUpload-service.tsx` sends documents in `FormData` under the `document` key.
- `auth.tsx` sends signup confirmation data as JSON in a `request` multipart part and optionally appends `orgLogo`.

Rules:

- Always validate type and size before upload.
- Never trust the browser validation alone; backend validation must also exist.
- Do not log file contents, parsed document text, binary data, or raw upload payloads.
- Use `multipart/form-data` only for actual file uploads.
- Revoke object URLs after downloads/previews when created with `URL.createObjectURL`.

## Output Sanitization and Rendering

React escapes normal JSX text by default. Keep using JSX text rendering for user and API content.

High-risk pattern:

- `keyword-list-grid.tsx` and `group-list-grid.tsx` use `dangerouslySetInnerHTML` to render trusted icon strings from `src/assets/svg-service.tsx`.

Rules:

- Do not use `dangerouslySetInnerHTML` for API-provided content.
- If SVG/icon HTML must be rendered, only render trusted local constants.
- Render URLs through normal `href`/`src` attributes after validation or protocol normalization.
- `prefixUrl.tsx` changes URL protocol based on `VITE_UPSTREAM`; validate external URLs before adding new uses.

## Error Handling and Logging

Existing code often logs errors with `console.error` and shows a friendlier modal. New code should tighten this:

- User-facing messages should be generic where sensitive details may be present.
- Detailed backend errors, token values, raw auth responses, uploaded document data, and profile data should not be logged.
- Do not log `localStorage.auth`, access tokens, refresh tokens, OTP values, passwords, uploaded file contents, full document summaries, or payment checkout responses.
- Avoid logging environment values in production. `prefixUrl.tsx` currently logs config values; do not add more.

Examples to avoid copying:

- `SignInView.tsx` logs submitted login data.
- `AuthContext.tsx` logs refresh response and token timing.
- `LandingPageScreen.tsx` logs document processing states.
- `package.json` hardcodes a scanner token.

When touching these areas, prefer reducing sensitive logging without changing unrelated behavior.

## Frontend/Backend Boundary

Frontend responsibilities:

- Gather input.
- Validate for UX.
- Attach bearer token.
- Show safe loading/success/error states.
- Hide or disable UI for unauthenticated, non-admin, or non-paid users.

Backend responsibilities:

- Enforce authentication and authorization.
- Validate payloads and uploaded files.
- Enforce quotas and subscriptions.
- Protect files and generated reports.
- Sanitize stored content.
- Keep secrets private.

Do not implement security solely through UI controls.

## Dependency and Build Security

Known libraries involved in sensitive flows:

- `axios` for API calls.
- `@tanstack/react-query` for data fetching.
- `react-hook-form` and `zod` for validation.
- `react-dropzone` and `pdfjs-dist` for upload handling.
- `@stomp/rx-stomp` and `rxjs` for progress messages.
- `redux-persist` for persisted Redux state.

Rules:

- Keep dependencies pinned through `package-lock.json`.
- Do not add a new auth, storage, upload, or sanitizer dependency without checking existing utilities first.
- Run `npm run build` after security-sensitive TypeScript changes when feasible.
- Treat `npm run lint` as uncertain until `eslint.config.js` is verified because the inspected file appears commented out.

## Security Checklist For Code Changes

- Does the change read, write, or log `auth`, `accessToken`, `refreshToken`, `email`, `userId`, profile data, payment data, uploaded files, or document summaries?
- Are authenticated API calls using `createAxiosInstance` unless there is a clear reason not to?
- Are tokens sent in headers, not query strings?
- Are new env values browser-safe if prefixed with `VITE_`?
- Are secrets kept out of source, env files, and `package.json`?
- Are user inputs validated with React Hook Form and Zod where this project already uses them?
- Are uploaded files checked for type and size before being sent?
- Are user-facing errors safe and non-sensitive?
- Is any use of `dangerouslySetInnerHTML` limited to trusted local icon constants?
- Are authorization checks enforced by the backend, with frontend checks only improving UX?
- Did you avoid adding console logs for raw requests, responses, tokens, OTPs, passwords, or files?

## Common Mistakes To Avoid

- Copying hardcoded URLs from `common.tsx`.
- Adding new manual bearer-token parsing in every service.
- Trusting `isAdmin`, `isPaid`, disabled buttons, or `ProtectedComponent` as real security.
- Storing extra sensitive state in `localStorage`.
- Showing raw `error.response.data` directly to users.
- Logging login form data or token refresh responses.
- Rendering backend-provided HTML.
- Adding private keys to `VITE_*` variables.

## Unclear Areas

- The exact backend authorization model is not visible from this frontend repository.
- `role` is removed on logout but not consistently shown as the source of frontend authorization; Redux `user.role` appears to be the active source for `isAdmin`.
- Some legacy files use `VITE_API_URL` or `VITE_BASE_URI2`; confirm active backend routing before modifying them.
- There is no visible centralized sanitizer or security logging policy in this checkout.
