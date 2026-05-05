---
name: configuration
description: Manage LeaseDrop frontend configuration using the repository's Vite mode, environment variable, Axios, proxy, and deployment conventions.
---

# Configuration

Use this skill when adding configuration to this LeaseDrop frontend or reproducing the same configuration style in a new Vite/React project.

## Configuration Locations

- `.env`: present but empty in this checkout.
- `.env.dev`: development mode values for `npm run dev`.
- `.env.lo`: local mode values for `npm run lo`.
- `.env.uat`: UAT values for `npm run uat`.
- `.env.prod`: production values for `npm run build`.
- `vite.config.ts`: loads mode-specific env with `loadEnv(mode, process.cwd(), '')`, configures React, Tailwind, sourcemap behavior, dev server host, and proxies.
- `package.json`: scripts map modes to commands: `dev`, `lo`, `uat`, `build`, `lint`, `verify`, and `preview`.
- `tailwind.config.js`: design tokens such as `leaseDrop_blue`, `leaseDrop_black`, `primary`, `danger`, and custom font families.
- `JenkinsfileProd`, `JenkinsfileUat`, and `Jenkinsfilelanding`: CI build/deploy entry points.
- `Env-scale.md`: architecture notes for upload processing, Azure Blob, Kafka, PostgreSQL, and WebSocket notification flow.

## Package Version Baseline

Future LeaseDrop-style projects should start from the same direct package set and version specs used in this app. Copy these entries from `package.json` and keep `package-lock.json` committed so transitive dependency versions resolve consistently.

Important rules:

- Do not casually upgrade packages when creating a similar project. Use the exact package names and version specs below unless the task is explicitly a dependency upgrade.
- Preserve `package-lock.json` for reproducible installs. `package.json` contains semver ranges such as `^18.3.1`; the lockfile records the exact resolved dependency tree.
- The local `file:` dependencies, `bef-lease-drop` and `pdf-rid`, are project-specific local package references. Confirm their source before reusing them in a new repository.
- If a future project must change a version, document why and test the affected feature area, especially routing, forms, React Query grids, PDF rendering, upload, and auth.

Runtime dependencies from this app:

```json
{
  "@hookform/resolvers": "^4.1.3",
  "@preact/signals-react": "^1.3.8",
  "@radix-ui/react-popover": "^1.1.4",
  "@radix-ui/react-tooltip": "^1.1.6",
  "@react-pdf-viewer/core": "^3.12.0",
  "@react-pdf-viewer/default-layout": "^3.12.0",
  "@react-pdf-viewer/search": "^3.12.0",
  "@reduxjs/toolkit": "^2.8.2",
  "@stomp/rx-stomp": "^2.0.1",
  "@tailwindcss/postcss": "^4.0.17",
  "@tailwindcss/vite": "^4.0.17",
  "@tanstack/react-query": "^5.70.0",
  "axios": "^1.7.7",
  "bef-lease-drop": "file:",
  "class-variance-authority": "^0.7.1",
  "clsx": "^2.1.1",
  "dotenv": "^16.4.7",
  "framer-motion": "^12.12.1",
  "fuse.js": "^7.1.0",
  "iconsax-react": "^0.0.8",
  "lottie-react": "^2.4.1",
  "lucide-react": "^0.485.0",
  "pdf-rid": "file:",
  "pdfjs-dist": "^3.4.120",
  "react": "^18.3.1",
  "react-datepicker": "^8.2.0",
  "react-day-picker": "^9.6.5",
  "react-dnd": "^16.0.1",
  "react-dnd-html5-backend": "^16.0.1",
  "react-dom": "^18.3.1",
  "react-dropzone": "^14.3.5",
  "react-hook-form": "^7.53.2",
  "react-intersection-observer": "^9.16.0",
  "react-pdf": "^9.2.1",
  "react-redux": "^9.2.0",
  "react-router-dom": "^6.27.0",
  "react-spinners": "^0.17.0",
  "react-stomp": "^5.1.0",
  "react-toastify": "^11.0.5",
  "react-tooltip": "^5.28.0",
  "react-window": "^2.2.1",
  "redux-persist": "^6.0.0",
  "rxjs": "^7.8.2",
  "screenfull": "^6.0.2",
  "tailwind-merge": "^2.6.0",
  "tailwindcss": "^4.0.17",
  "world-countries": "^5.1.0",
  "zod": "^3.24.2"
}
```

Development dependencies from this app:

```json
{
  "@eslint/js": "^9.12.0",
  "@types/node": "^22.10.5",
  "@types/react": "^18.3.2",
  "@types/react-dom": "19.0.0",
  "@typescript-eslint/eslint-plugin": "^8.8.1",
  "@typescript-eslint/parser": "^8.8.1",
  "@vitejs/plugin-react": "^4.3.2",
  "eslint": "^9.12.0",
  "eslint-config-prettier": "^9.1.0",
  "eslint-plugin-import": "^2.31.0",
  "eslint-plugin-jsx-a11y": "^6.10.0",
  "eslint-plugin-prettier": "^5.2.1",
  "eslint-plugin-react": "^7.37.1",
  "eslint-plugin-react-hooks": "^5.1.0-rc.0",
  "eslint-plugin-react-refresh": "^0.4.12",
  "globals": "^15.11.0",
  "prettier": "3.3.3",
  "typescript": "^5.6.3",
  "typescript-eslint": "^8.8.1",
  "vite": "^5.4.8"
}
```

Package roles in this codebase:

- React app foundation: `react`, `react-dom`, `vite`, `@vitejs/plugin-react`, `typescript`.
- Routing: `react-router-dom`.
- API and server state: `axios`, `@tanstack/react-query`.
- Auth and persisted app state: `@reduxjs/toolkit`, `react-redux`, `redux-persist`.
- Forms and validation: `react-hook-form`, `@hookform/resolvers`, `zod`.
- UI styling and class utilities: `tailwindcss`, `@tailwindcss/vite`, `@tailwindcss/postcss`, `tailwind-merge`, `clsx`, `class-variance-authority`.
- UI primitives and icons: `@radix-ui/react-popover`, `@radix-ui/react-tooltip`, `lucide-react`, `iconsax-react`.
- PDF/document features: `pdfjs-dist`, `react-pdf`, `@react-pdf-viewer/core`, `@react-pdf-viewer/default-layout`, `@react-pdf-viewer/search`, `pdf-rid`.
- Upload, date, drag/drop, and utility UI: `react-dropzone`, `react-datepicker`, `react-day-picker`, `react-dnd`, `react-dnd-html5-backend`, `react-spinners`, `react-tooltip`, `react-toastify`, `react-window`, `screenfull`.
- Realtime/progress flow: `@stomp/rx-stomp`, `react-stomp`, `rxjs`.
- Motion/search/content helpers: `framer-motion`, `fuse.js`, `lottie-react`, `world-countries`, `@preact/signals-react`.

## Environment Variables Used

The app reads Vite-exposed environment variables through `import.meta.env`. Existing variable names include:

- `VITE_BASE_URI`: primary backend API base URL. Used by most API services and `createAxiosInstance`.
- `VITE_BASE_URI2`: secondary backend API base URL. It appears in several files but is often unused or legacy.
- `VITE_WEBSOCKET_URI`: WebSocket/STOMP base URL. `LandingPageScreen.tsx` builds `${VITE_WEBSOCKET_URI}/leasedrop-session`.
- `VITE_UPSTREAM`: protocol switch used by `src/views/utils/prefixUrl.tsx`; when it equals `'TRUE'`, generated file URLs use `https`, otherwise `http`.
- `VITE_API_URL`: referenced by old files such as `src/services/api/contact.tsx` and `src/services/api/dropdown.tsx`, but not present in the inspected `.env.*` variable lists. Treat this as unclear/legacy unless you confirm it is required.

Only variables prefixed with `VITE_` are available to browser code. Do not add unprefixed frontend variables expecting them to work in React components.

## Mode Handling

Existing scripts:

```json
{
  "dev": "vite --mode dev",
  "lo": "vite --mode lo",
  "uat": "tsc -b && vite build --mode uat",
  "build": "tsc -b && vite build --mode prod"
}
```

Vite loads `.env.<mode>` for each script. `vite.config.ts` also loads env values for dev-server proxy targets:

```ts
const env = loadEnv(mode, process.cwd(), '');
proxy: {
  '/api': {
    target: env.VITE_BASE_URI,
    changeOrigin: true,
    rewrite: (path) => path.replace(/^\/api/, ''),
  }
}
```

For new modes, add a matching script and `.env.<mode>` file, then confirm Vite and Jenkins use the same mode name.

## API URL Configuration Pattern

Feature services usually read the base URL once at module scope:

```ts
const baseUrl: string = import.meta.env.VITE_BASE_URI || '';
const axiosInstance = createAxiosInstance(baseUrl);
```

Then services build endpoint paths from the base URL or pass relative paths to the configured Axios instance.

Preferred pattern for new authenticated services:

```ts
import { AxiosResponse } from 'axios';
import createAxiosInstance from '../../config/axios-config';

const baseUrl: string = import.meta.env.VITE_BASE_URI || '';
const axiosInstance = createAxiosInstance(baseUrl);

export function useExampleService() {
  const exampleBaseUrl = '/examples';

  function getExampleList(searchBody: any): Promise<AxiosResponse> {
    return axiosInstance.post(`${exampleBaseUrl}/search`, searchBody);
  }

  return { getExampleList };
}
```

Avoid the legacy pattern in `src/services/api/common.tsx`, which hardcodes an internal URL. That file is an exception, not a convention to copy.

## Constants and Feature Flags

- Tailwind theme constants belong in `tailwind.config.js`.
- API endpoint bases belong in feature service files.
- WebSocket constants such as `PRV_PREFIX`, `SUB_USER_MESSAGE_PRV_URL`, and `PUB_APP_USERS` currently live in `LandingPageScreen.tsx`.
- `VITE_UPSTREAM` is used as a simple environment flag. Keep flag values explicit and documented, because the current implementation compares to the string `'TRUE'`.

## Security Rules

- Do not commit secrets. This project currently contains a hardcoded Sonar token in the `package.json` `verify` script and several committed `.env.*` files. Treat those as security debt, not precedent.
- Do not add API keys, tokens, credentials, or private URLs to source files.
- Do not expose secrets through `VITE_` variables. Anything prefixed `VITE_` is bundled into browser-accessible code.
- Keep deploy credentials in Jenkins credentials or another CI secret store, following the `GITLAB_ID = 'gitlab-credentials'` style used in Jenkinsfiles.
- Avoid logging env-derived URLs in production code. `prefixUrl.tsx` currently logs `VITE_UPSTREAM`; new code should not add more config logging.

## Required Configuration Validation

The current project usually falls back to an empty string when config is missing:

```ts
const baseUrl: string = import.meta.env.VITE_BASE_URI || '';
```

This avoids immediate crashes but can produce broken runtime URLs. For new projects using this style, add a small config module that validates required frontend config once:

```ts
type AppConfig = {
  baseUri: string;
  websocketUri: string;
  upstream: 'TRUE' | 'FALSE';
};

const required = (name: string) => {
  const value = import.meta.env[name];
  if (!value) throw new Error(`Missing required environment variable: ${name}`);
  return value;
};

export const appConfig: AppConfig = {
  baseUri: required('VITE_BASE_URI'),
  websocketUri: required('VITE_WEBSOCKET_URI'),
  upstream: (import.meta.env.VITE_UPSTREAM || 'FALSE') as 'TRUE' | 'FALSE',
};
```

If adding this to the existing project, migrate gradually and avoid changing every service in one unrelated feature branch.

## Recommended Template For Similar Projects

Root env files:

```dotenv
# .env.dev
VITE_BASE_URI=https://dev-api.example.com
VITE_WEBSOCKET_URI=wss://dev-ws.example.com
VITE_UPSTREAM=TRUE
```

```dotenv
# .env.uat
VITE_BASE_URI=https://uat-api.example.com
VITE_WEBSOCKET_URI=wss://uat-ws.example.com
VITE_UPSTREAM=TRUE
```

```dotenv
# .env.prod
VITE_BASE_URI=https://api.example.com
VITE_WEBSOCKET_URI=wss://ws.example.com
VITE_UPSTREAM=TRUE
```

Rules:

- Keep non-secret public runtime values in `.env.<mode>`.
- Keep secrets out of frontend env files entirely.
- Keep deployment secrets in CI/CD secret stores.
- Keep API endpoint paths in service files, not in components.
- Keep backend host/base URL in env, not hardcoded strings.

## Checklist For Adding Configuration

- Confirm whether the value is safe to expose in browser code.
- If browser-safe, add it as `VITE_*` to every required `.env.<mode>` file.
- Update `vite.config.ts` only if Vite itself needs the value, such as proxy configuration.
- Read it through `import.meta.env` or a config module.
- Do not duplicate the value in components, services, and Jenkinsfiles.
- Add validation for values required at app startup.
- Avoid logging config values unless needed temporarily for local debugging.
- Check `npm run dev`, `npm run lo`, `npm run uat`, and `npm run build` implications before changing mode names.

## Unclear Areas

- `VITE_BASE_URI2` is declared and imported in several services but is often unused.
- `VITE_API_URL` is referenced in a few older files but was not found in the inspected env variable names.
- `.env.dev` is ignored by `.gitignore`, but env files are present in the checkout. Confirm team policy before adding or removing env files from source control.
- `eslint.config.js` appears to contain only commented configuration in this checkout, so `npm run lint` may not represent an active lint policy until verified.
