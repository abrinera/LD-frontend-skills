---
name: react-security-hardening
description: >
  Apply this skill whenever the user asks about security in a React + TanStack + shadcn/ui codebase.
  Triggers include: rate limiting, hardcoded secrets/tokens/API keys, environment variables, input
  sanitization, security audits, XSS, CSRF, auth vulnerabilities, exposed credentials, or any request
  to "scan", "audit", "harden", or "secure" the codebase. Also triggers for prompts like "move secrets
  to env", "sanitize inputs", "check for vulnerabilities", or "limit login attempts". Use this skill
  even if the user only mentions one of these concerns — run the full relevant section.
---

# React Security Hardening Skill

Stack: **React · TanStack (Query / Router / Form) · shadcn/ui · Node/Express or Next.js API routes**

---

## Mental Model

Think in layers. Every fix belongs to exactly one layer. Never mix them.

```
Browser (React)      →  Sanitize inputs, never store secrets, validate client-side only as UX
Network (API routes) →  Rate limit, auth checks, CSRF, size limits
Server / Env         →  Secrets live here only — never in code, never in frontend bundles
Audit                →  Scan all layers, report by severity
```

---

## 1 · Rate Limiting on Login (max 5 / 15 min)

**Scope**: API route only — never implement rate limiting in React components.

### Node / Express
```ts
// lib/rateLimit.ts
import rateLimit from "express-rate-limit";

export const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,   // 15 minutes
  max: 5,
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: "Too many login attempts. Try again in 15 minutes." },
  // Persist across restarts in production:
  // store: new RedisStore({ client })
});

// routes/auth.ts
router.post("/login", loginLimiter, loginHandler);
```

### Next.js App Router
```ts
// app/api/auth/login/route.ts
import { NextRequest, NextResponse } from "next/server";
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(5, "15 m"),
});

export async function POST(req: NextRequest) {
  const ip = req.headers.get("x-forwarded-for") ?? "anonymous";
  const { success, reset } = await ratelimit.limit(`login:${ip}`);

  if (!success) {
    return NextResponse.json(
      { error: "Too many attempts. Try again later." },
      {
        status: 429,
        headers: { "Retry-After": String(Math.ceil((reset - Date.now()) / 1000)) },
      }
    );
  }
  // ... login logic
}
```

**Checklist**
- [ ] Rate limit keyed by IP (+ user identifier when available, to prevent credential stuffing)
- [ ] Uses Redis/persistent store in production (in-memory resets on deploy)
- [ ] Returns `429` with `Retry-After` header
- [ ] Login UI shows remaining wait time (read from `Retry-After` response header)

---

## 2 · Scan for Hardcoded Secrets

Run these commands from the project root before every audit:

```bash
# --- Quick scan (no install needed) ---
grep -rn \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" --include="*.env*" \
  -E "(api_key|apikey|api_secret|secret_key|password|token|bearer|private_key|PRIVATE_KEY)\s*[:=]\s*['\"][^'\"]{8,}" \
  . | grep -v "node_modules" | grep -v ".next"

# --- Scan for raw URLs with embedded credentials ---
grep -rn --include="*.ts" --include="*.tsx" \
  -E "https?://[^@\s]+:[^@\s]+@" . | grep -v "node_modules"

# --- Check .env files are gitignored ---
git check-ignore -v .env .env.local .env.production

# --- Audit git history for accidental commits ---
git log --all --full-history -- "*.env" "*.pem" "*secret*" "*token*"
```

**Red flags to look for manually**
| Pattern | Risk |
|---|---|
| `const API_KEY = "sk-..."` | Critical — exposed in bundle |
| `fetch("https://api.x.com?key=abc123")` | Critical — visible in network tab |
| `Authorization: "Bearer hardcoded"` | Critical |
| `password: "dev-password"` in seed files committed | High |
| `NEXT_PUBLIC_SECRET_KEY` | High — `NEXT_PUBLIC_` prefixes ship to browser |

---

## 3 · Move Secrets to Environment Variables

### Rules
1. **Never prefix secrets with `NEXT_PUBLIC_`** — that ships them to the browser bundle.
2. **All secrets in `.env.local`** (dev) / Vercel / Railway / Doppler (prod). Never `.env` committed to git.
3. **Validate env vars at startup** — fail loud, fail early.

### Env structure
```
.env.local          ← dev secrets (gitignored)
.env.example        ← committed, all keys present, all values blank or fake
.gitignore          ← must include: .env .env.local .env*.local
```

### Startup validation (add to `lib/env.ts`)
```ts
import { z } from "zod";

const envSchema = z.object({
  DATABASE_URL:      z.string().url(),
  JWT_SECRET:        z.string().min(32),
  STRIPE_SECRET_KEY: z.string().startsWith("sk_"),
  // Public vars (safe to expose):
  NEXT_PUBLIC_APP_URL: z.string().url(),
});

export const env = envSchema.parse(process.env);
// Import `env` everywhere instead of process.env directly.
// App crashes at boot if any secret is missing — better than silent failures at runtime.
```

### Migrate a hardcoded secret
```ts
// BEFORE ❌
const stripe = new Stripe("sk_live_abc123...");

// AFTER ✅
import { env } from "@/lib/env";
const stripe = new Stripe(env.STRIPE_SECRET_KEY);
```

**Checklist**
- [ ] `.env.example` committed with blank values
- [ ] `.env.local` and `.env.production` in `.gitignore`
- [ ] `lib/env.ts` validates all secrets at boot
- [ ] No `NEXT_PUBLIC_` prefix on any secret
- [ ] CI/CD secrets stored in platform secret manager, not in repo

---

## 4 · Sanitize User Inputs

### Layer 1 — Client (UX only, never trust this)
Use TanStack Form + Zod. This is convenience, not security.

```ts
// form/LoginForm.tsx — example schema
import { z } from "zod";

export const loginSchema = z.object({
  email:    z.string().email().max(254),
  password: z.string().min(8).max(128),
});
```

### Layer 2 — API route (authoritative validation)
```ts
// app/api/auth/login/route.ts
import { loginSchema } from "@/lib/schemas/auth";

export async function POST(req: NextRequest) {
  const body = await req.json().catch(() => null);

  if (!body) {
    return NextResponse.json({ error: "Invalid JSON" }, { status: 400 });
  }

  const parsed = loginSchema.safeParse(body);
  if (!parsed.success) {
    return NextResponse.json(
      { error: "Validation failed", issues: parsed.error.flatten() },
      { status: 422 }
    );
  }
  // Use parsed.data only — never raw body
}
```

### Layer 3 — Size limits (reject oversized payloads)
```ts
// middleware.ts or express middleware
const MAX_BODY_SIZE = 10 * 1024; // 10 KB

export async function middleware(req: NextRequest) {
  const contentLength = Number(req.headers.get("content-length") ?? 0);
  if (contentLength > MAX_BODY_SIZE) {
    return NextResponse.json({ error: "Payload too large" }, { status: 413 });
  }
}
```

### Layer 4 — Output sanitization (XSS)
```ts
// Never use dangerouslySetInnerHTML with user content.
// If you must render user HTML (e.g. rich text), sanitize first:
import DOMPurify from "dompurify";

const clean = DOMPurify.sanitize(userHtml, { ALLOWED_TAGS: ["b", "i", "em", "strong"] });
<div dangerouslySetInnerHTML={{ __html: clean }} />
```

**Checklist**
- [ ] Zod schema on every API route — no raw `req.body` access
- [ ] `MAX_BODY_SIZE` enforced at middleware level
- [ ] File uploads: check MIME type server-side, not just extension
- [ ] No `dangerouslySetInnerHTML` without DOMPurify
- [ ] SQL/ORM queries use parameterized inputs only (no string concatenation)

---

## 5 · Full Security Audit — Report Format

When running a full audit, scan all five layers and produce output in this structure:

```
## Security Audit Report
Generated: <date>
Stack: React + TanStack + shadcn/ui

### CRITICAL (fix before deploy)
[C1] <file>:<line> — <description> — Recommendation: <fix>

### HIGH (fix this sprint)
[H1] <file>:<line> — <description> — Recommendation: <fix>

### MEDIUM (schedule fix)
[M1] ...

### LOW / INFO
[L1] ...

### PASSED CHECKS ✅
- Rate limiting on /api/auth/login
- .env.local gitignored
- ...

### Summary
Critical: N | High: N | Medium: N | Low: N
```

**Audit checklist — run in order**

```bash
# 1. Dependency vulnerabilities
npm audit --audit-level=moderate

# 2. Hardcoded secrets (see Section 2 commands)

# 3. Exposed env vars in bundle
grep -rn "NEXT_PUBLIC_" .env* --include="*.ts" --include="*.tsx"

# 4. Missing auth checks on API routes
grep -rL "getServerSession\|auth()\|verifyToken\|requireAuth" app/api/**/route.ts

# 5. dangerouslySetInnerHTML usage
grep -rn "dangerouslySetInnerHTML" src/ app/ --include="*.tsx"

# 6. console.log with sensitive data
grep -rn "console.log" --include="*.ts" --include="*.tsx" . | grep -i "token\|password\|secret\|key"

# 7. HTTP (non-HTTPS) URLs in production code
grep -rn "http://" src/ app/ --include="*.ts" --include="*.tsx" | grep -v "localhost\|127.0.0.1\|// "
```

**Manual checks**
- [ ] HTTPS enforced on all routes (no mixed content)
- [ ] Auth tokens stored in `httpOnly` cookies — not `localStorage`
- [ ] CSRF protection on all state-mutating routes
- [ ] Error messages don't leak stack traces or DB schema to client
- [ ] Dependency audit passes with no critical/high CVEs
- [ ] `Content-Security-Policy` header set
- [ ] `X-Frame-Options: DENY` header set

---

## Quick Reference — Security Decision Tree

```
User input arrives at API →
  ├── Body size OK?          → 413 if not
  ├── JSON parseable?        → 400 if not
  ├── Zod schema passes?     → 422 if not
  ├── Rate limit OK?         → 429 if not (login routes)
  ├── Auth token valid?      → 401 if not
  └── Authorized for action? → 403 if not
      └── Process with parsed.data ✅
```

---

## Files This Skill Touches

| File | Purpose |
|---|---|
| `lib/env.ts` | Typed, validated env vars |
| `lib/rateLimit.ts` | Reusable rate limiter |
| `lib/schemas/*.ts` | Shared Zod schemas (used by both form and API) |
| `middleware.ts` | Size limits, auth checks, CSP headers |
| `.env.example` | Committed template — no real values |
| `.gitignore` | Must include `.env*` patterns |
