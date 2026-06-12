---
name: preprod-check
description: >-
  Use this skill when the user asks for a pre-production readiness check,
  launch checklist, "is this safe to ship", "go-live review", "prod audit",
  or any variant of "what should I check before deploying". Performs a
  structured audit covering auth/multi-tenancy, input validation, billing
  & credit integrity, rate limiting, cost containment, external-request
  safety (SSRF, uploads), secrets/env, security headers/cookies, error
  handling, CORS, database (indexes/backups), logging & monitoring,
  email/password flows, legal/compliance, and operations. Adapts checks
  to the detected stack (Next.js, Auth.js, Stripe, Supabase, Drizzle,
  AI SDKs, blob storage, etc.). Produces a severity-grouped findings
  report and drafts patches for trivial fixes. Also invocable as
  `/preprod-check`.
metadata:
  author: kevincui1034
  version: "1.0.0"
---

# Pre-Prod Readiness Check

Audit a project for production readiness across 14 categories. Adapt to the detected stack. Produce a **severity-grouped** report with file:line refs, then propose patches for trivial fixes (security headers, cookie flags, env schema, etc.) for user approval.

This skill assumes a TypeScript/Next.js web app by default but degrades gracefully — skip categories that don't apply to the detected stack (e.g., skip "billing & credit integrity" if there's no Stripe).

## How to use this skill

Run the **workflow** in order. Each step has a purpose; don't skip the discovery phase or the checks become generic.

The **check catalog** at the bottom is the source of truth for what to inspect — refer to it during step 2, but report findings using the severity rubric, not by catalog category.

---

## Workflow

### Step 1 — Stack detection (parallel reads)

Read these in parallel, then write a one-paragraph stack summary before doing any checks:

- `package.json` — frameworks, AI SDKs, payment SDKs, ORM, auth libs
- `next.config.*` / `vite.config.*` / similar — framework config
- `tsconfig.json` — strict mode, path aliases
- `CLAUDE.md` and `AGENTS.md` (if present) — project conventions
- `.env.example` (if present) — declared env surface
- `src/lib/db/schema.*` or `prisma/schema.prisma` or `drizzle.config.*` — DB shape
- `src/proxy.ts` / `src/middleware.ts` / `src/auth.*` — auth wiring
- `src/app/api/**` or `pages/api/**` directory listing — API surface (use Glob, don't read all)

**Detect**:
- Framework + version (Next.js 16 vs 15 differs; React Router vs Next; etc.)
- Auth library (Auth.js v5 / Clerk / Lucia / custom)
- Payment processor (Stripe / Lemon Squeezy / none)
- DB (Postgres via Supabase / Neon / Vercel; MySQL; SQLite) + ORM (Drizzle / Prisma / Kysely)
- AI / external APIs (Anthropic, Gemini, OpenAI, Replicate, Fal, Apify, Resend)
- File storage (Vercel Blob, S3, R2)
- Hosting target (Vercel, Cloudflare, self-hosted)

Skip catalog categories that don't apply. E.g., no Stripe → skip §3.

### Step 2 — Run checks (catalog below)

Walk the **check catalog** below. For each applicable check:
1. Use Grep / Read / Glob to locate the relevant code.
2. Where the catalog calls for a diagnostic command, run it (npm audit, lint, env grep). Don't run network calls or destructive commands.
3. Record findings with `file:line` and a one-line description.
4. Assign severity using the rubric.

Be evidence-driven. Don't report a "missing CSP" as a finding if you haven't actually grepped for `Content-Security-Policy` and confirmed it's absent.

### Step 3 — Severity rubric

| Severity | Meaning | Examples |
| --- | --- | --- |
| **Critical** | Money loss, data leak, or auth bypass possible today | Webhook handler skips idempotency insert; credit ledger has no concurrency guard; server action missing `auth()`; SSRF on a URL fetcher; secret in client bundle |
| **High** | Realistic incident path; not actively bleeding | No per-user $ ceiling; no CSP; mime not sniffed on uploads; password reset token has no expiry; missing env validation at boot |
| **Medium** | Hygiene gap; would slow incident response or DX | Missing structured logs; no Sentry filter; FK column lacking index; no health check endpoint |
| **Low** | Polish; would be embarrassing but not damaging | 500 page leaks stack trace; missing privacy policy link; cookie flags relying on framework defaults rather than explicit |

When in doubt, **err one level higher** — a single false-positive Critical is cheaper than missing a real one.

### Step 4 — Output the report

Output in this exact structure (markdown). Group by severity, not by catalog category. Each finding is one bullet:

```
## Pre-Prod Findings — <project name> (<date>)

**Stack detected**: <one-line summary>
**Categories checked**: <comma-separated, with skipped ones in parens>

### Critical (N)
- [file.ts:42](src/file.ts#L42) — <what's wrong>. Fix: <one-line sketch>.

### High (N)
- ...

### Medium (N)
- ...

### Low (N)
- ...

### Skipped / not applicable
- <category>: <reason>

### Patches I can draft now
- <finding> — would touch <file:line>
- ...
```

Keep each finding to **one line**. Detail belongs in the patch, not the report.

### Step 5 — Propose patches

After the report, ask which findings to patch. For each approved one:
- Show a unified diff or a complete file rewrite for new files.
- Stick to the trivial / mechanical fixes from the **Patch templates** section. Don't draft business-logic changes (e.g., credit ledger redesign) — flag those as needing a follow-up plan.

**Always** wait for explicit approval before editing. Don't batch — one finding, one patch, one approval.

---

## Check catalog

Each section: what to grep, what counts as a finding, default severity.

### 1. Authentication & multi-tenant isolation

- **Every server action / route handler begins with an auth check.** Grep `server-only` modules and `actions.ts` files for the pattern `await auth()` (or equivalent). Any action missing it → **Critical**.
- **Cross-tenant joins re-verify ownership server-side.** Grep for FK columns that come from request input (`userId`, `projectId`, `personaId` etc.) and trace whether the handler re-scopes to `session.user.id` before using them. Missing scope → **Critical**.
- **JWT/session callbacks refresh sensitive fields per-request.** If plan, ban status, or session-invalidation timestamps are mirrored on the session, the callback must re-read them. Stale session = stale plan / unrevoked sessions. **High**.
- **Middleware/proxy and layout both gate auth.** Defense-in-depth — both should redirect unauthed. Single layer → **Medium**.
- **Password reset bumps session-invalidation timestamp** to revoke other sessions. Missing → **High**.

### 2. Input validation & injection

- **Zod (or similar) on every server action input.** Grep for `formData.get(` followed by direct DB writes without a schema parse. Unvalidated input flowing to DB → **High**.
- **Parameterized queries everywhere.** Grep for template-string SQL (`` `SELECT ... ${...}` ``) outside of `sql` tagged templates. Raw interpolation → **Critical**.
- **No `dangerouslySetInnerHTML`** on user-supplied content without DOMPurify (or equivalent). **High**.
- **Path traversal on file ops.** Anywhere `fs.readFile` / `fs.writeFile` / Blob keys derive from user input — must `path.normalize` and reject `..`. **High**.
- **Redirect URL validation.** Open redirects via `?next=` or `?callbackUrl=` must whitelist host. **High**.

### 3. Billing & credit integrity (skip if no Stripe / no in-app currency)

- **Webhook signature verification before any DB write.** Grep the webhook route for `stripe.webhooks.constructEvent` (or processor equivalent) — must happen before parsing body. **Critical** if missing.
- **Webhook idempotency table.** Every handler inserts the event ID first and bails on conflict. Grep webhook handlers for an idempotency insert. Missing → **Critical**.
- **Credit ledger concurrency guard.** A `SELECT … FOR UPDATE`, optimistic version column, or DB-level `CHECK (balance >= 0)`. Without it, parallel debits can go negative. **Critical**.
- **Refund-on-failure audit.** For each external-API call charged to credits, trace every error branch — every one must refund. List any path that doesn't. **Critical** per missing branch.
- **Monthly grant uses `GREATEST(balance, allotment)`** (not blind set) so top-up purchases survive grants. **High** if blind set.
- **Provider-specific pricing tables stay in sync.** If credit costs live in one file and USD-per-unit in another, drift = margin loss. Flag tuples present in one but missing in the other. **High**.

### 4. Rate limiting & abuse

- **Per-user daily $ ceiling** on AI/LLM/render spend, independent of credit balance and quota. Catches runaway loops and prompt-injection cost attacks. Missing → **High**.
- **Quota gate on every external API call** the user triggers. Grep for `fetch(` to known AI/scraper hosts in server modules and verify each is preceded by a quota or credit check. **High** per gap.
- **IP rate limit on signup / login / password reset.** Upstash Ratelimit, Vercel KV, or similar. Missing on auth endpoints → **High**.
- **Webhook endpoints aren't rate-limited the same way** (signature is the gate), but verify they reject unsigned requests fast (cheap 401 before any work). **Medium**.

### 5. Cost containment & external API safety

- **Timeouts on every external call.** Grep for `fetch(` without `AbortSignal.timeout` or equivalent. Default timeout < platform function ceiling. **High** per uncapped call.
- **Cost anomaly alerts.** Some mechanism (Sentry, PostHog, cron) that flags > N× rolling-median daily spend. Missing → **Medium**.
- **Circuit breaker / retry budget** on flaky providers. Missing isn't critical but means a provider outage can hang functions. **Low/Medium**.
- **`maxDuration` / `memory` config on long-running routes.** Mismatch between actual work and config → silent timeouts. **Medium**.

### 6. External-request safety (SSRF, uploads, blob URLs)

- **SSRF blocklist on any URL the server fetches from user input.** Block: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `169.254.0.0/16` (cloud metadata!), `::1`, `fc00::/7`. Resolve DNS first, then check. **Critical** if missing on user-supplied URL fetchers.
- **Upload mime sniffing.** Server reads first bytes; doesn't trust `Content-Type`. **High** if absent.
- **Size caps enforced server-side** (not just client). Grep for `maxSize` / content-length checks on upload routes. **High** if client-only.
- **Blob/storage URL host validation** on routes that accept URLs back from the upload service. Pin to your bucket/blob host. **High**.

### 7. Secrets & env

- **No secrets in client bundle.** Grep `NEXT_PUBLIC_` (or framework equivalent) for anything that looks like a key/secret/token. **Critical** per leak.
- **Env validation at boot.** A zod (or t3-env) schema that throws on missing required vars at startup. **High** if absent.
- **`.env*` in `.gitignore`** (except `.env.example`). Verify with `git check-ignore`. **Critical** if `.env.local` is tracked.
- **Secrets aren't logged.** Grep `console.log` near auth/payment/AI modules for variables that match env names. **High** per leak.

### 8. Security headers & cookies

- **CSP, HSTS, `X-Frame-Options`, `Referrer-Policy`, `X-Content-Type-Options: nosniff`** set via middleware/proxy or `next.config.headers()`. Missing CSP → **High**; others → **Medium**.
- **Cookie flags**: `httpOnly`, `secure`, `sameSite=lax` (or `strict`) on session cookies. If relying on framework defaults, verify in prod build. **Medium**.
- **CORS** allowlist is explicit (not `*`) on any authed endpoint. **High** if wildcard.

### 9. Error handling & UX fallbacks

- **Error boundaries** (`error.tsx` in Next App Router, or framework equivalent) on every major route segment. Missing → **Medium**.
- **`not-found.tsx` / 404 page** exists. **Low** if missing.
- **No stack traces in user-facing responses.** Grep API route catches for `error.stack` or `error.message` returned to client. **High** if stack leaks; **Medium** if message leaks (depends on what message can contain).
- **Friendly fallback for quota / rate-limit / credit-exhausted errors** — these are expected, not exceptional. **Medium** if shown as generic "Something went wrong."

### 10. Database

- **Indexes on most-queried columns** — FK columns used in joins/filters, `created_at` if sorted on, anything in `WHERE` clauses of hot queries. Read the schema, grep top server modules for `.where(` / `.eq(` / etc. **Medium** per missing index on hot path.
- **Backups / PITR enabled** on the prod DB. Can't verify in code; surface as a manual check in the report. **Medium**.
- **Migrations are forward-only and reviewed.** No `db:push` against prod (dev-only command). **Low** unless evidence of misuse.
- **Sensitive tables** (users, sessions, payments) have proper `ON DELETE` semantics — `CASCADE` for owned rows, `RESTRICT` for audit logs. **Medium** per missing.

### 11. Logging & monitoring

- **Structured logs** (JSON) in prod, not just `console.log` strings. Either via a logger lib or `JSON.stringify`. **Medium** if absent.
- **Sentry / equivalent** configured with a `beforeSend` filter that drops expected user errors (quota exceeded, validation failures) so noise doesn't drown signal. **Medium**.
- **No PII in logs.** Grep logging calls for `email`, `phone`, full names, tokens. **High** per leak.
- **Health check endpoint** (`/api/health`) returns DB + critical-dependency status. **Low/Medium**.

### 12. Email & password

- **Password hashing**: argon2id or bcrypt with appropriate cost. Plain SHA, MD5, or any unhashed storage → **Critical**.
- **Password reset tokens**: single-use, expiry ≤ 1h, sent via signed link. Multi-use or no expiry → **High**.
- **SPF / DKIM / DMARC records** on the sender domain — can't verify from code; surface as a manual DNS check. **High** if production traffic, none configured.
- **Bounce/complaint handling** on transactional email provider (Resend, SES, Postmark) to avoid reputation damage. **Medium**.
- **Email verification required** before account-affecting actions (password change, etc.). **Medium**.

### 13. Legal & compliance

- **Privacy policy + ToS** linked from signup and footer. Required by Stripe, Google OAuth verification, app stores. Missing → **High** (blocks integrations) or **Low** (purely legal exposure) depending on context.
- **Account deletion path** that purges or anonymizes PII and cascades the FK graph. GDPR/CCPA. **Medium**.
- **Data export endpoint** (less urgent than deletion). **Low**.
- **Cookie consent** if serving EU traffic and using non-essential cookies. **Medium** for EU traffic.

### 14. Operations & supply chain

- **`npm audit` (or `pnpm audit --prod`)** — run it; report HIGH and CRITICAL vulns. **High** per critical CVE in a runtime dep.
- **Dependency pinning**: lockfile committed, no `^` ranges on security-critical libs in deploys. Lockfile is the main gate. **Low** if lockfile present.
- **Env parity**: a staging/preview env separate from prod. **Medium** if everything is one env.
- **CI/CD secrets** not echoed in logs. Can't always verify, but spot-check `.github/workflows/` for `echo $SECRET`-style patterns. **High** per leak.
- **No `--no-verify` / `--no-gpg-sign`** in committed scripts. **Low**.

---

## Patch templates

These are the trivial fixes the skill should be willing to draft. Each is a known-shape, mechanical change — not architectural.

### Security headers (Next.js 16)

In `next.config.ts` / `next.config.mjs`:

```ts
async headers() {
  return [{
    source: "/:path*",
    headers: [
      { key: "X-Frame-Options", value: "DENY" },
      { key: "X-Content-Type-Options", value: "nosniff" },
      { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
      { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" },
      { key: "Content-Security-Policy", value: "<draft a CSP based on the detected stack — be conservative; flag any inline scripts in the report>" },
    ],
  }];
}
```

For CSP, **don't draft a wildcard `unsafe-inline` policy** to make it "work" — list the actual hosts the app loads from (Stripe.js, Google fonts, analytics, etc.) based on grep findings.

### Env validation at boot

Drop a `src/env.ts`:

```ts
import "server-only";
import { z } from "zod";

const schema = z.object({
  DATABASE_URL: z.string().url(),
  AUTH_SECRET: z.string().min(32),
  // ...one line per required env var detected
});

export const env = schema.parse(process.env);
```

Then `import "@/env";` at the top of the auth/db entrypoints so it throws at boot.

### Stripe webhook idempotency (Drizzle example)

```ts
await db.insert(processedStripeEvent).values({ id: event.id }).onConflictDoNothing();
const inserted = await db.select().from(processedStripeEvent).where(eq(processedStripeEvent.id, event.id));
if (!inserted.length) return Response.json({ received: true }); // already processed
// ...then run the handler
```

Only draft this if the table doesn't already exist — if it does, audit the handlers instead.

### SSRF blocklist for URL fetchers

```ts
import { lookup } from "node:dns/promises";

const BLOCKED_CIDRS = [
  /^10\./, /^172\.(1[6-9]|2\d|3[01])\./, /^192\.168\./,
  /^127\./, /^169\.254\./, /^::1$/, /^fc/, /^fd/,
];

export async function assertPublicHost(url: string) {
  const { hostname } = new URL(url);
  const { address } = await lookup(hostname);
  if (BLOCKED_CIDRS.some(re => re.test(address))) {
    throw new Error("Refusing to fetch private/loopback address");
  }
}
```

Call `await assertPublicHost(url)` before any server-side fetch of user-supplied URLs.

### Upload mime sniffing

Use `file-type` (already common in the Node ecosystem):

```ts
import { fileTypeFromBuffer } from "file-type";

const head = Buffer.from(await blob.slice(0, 4100).arrayBuffer());
const sniffed = await fileTypeFromBuffer(head);
if (!sniffed || !ALLOWED_MIMES.has(sniffed.mime)) {
  return new Response("Unsupported file type", { status: 415 });
}
```

### Cookie flags (Auth.js v5)

If not relying on defaults, explicit in `auth.config.ts`:

```ts
cookies: {
  sessionToken: {
    name: "authjs.session-token",
    options: { httpOnly: true, secure: true, sameSite: "lax", path: "/" },
  },
},
```

### Sentry quota filter

```ts
Sentry.init({
  // ...
  beforeSend(event, hint) {
    const err = hint.originalException;
    if (err instanceof Error && /^Quota exceeded:/.test(err.message)) return null;
    return event;
  },
});
```

---

## Don'ts

- **Don't draft architectural patches.** A credit-ledger race fix is a design discussion, not a Write call. Flag it in the report and stop.
- **Don't run destructive or network-spending commands.** No `db:push`, no API calls that cost money, no `npm install` of new deps without approval.
- **Don't invent findings.** If you couldn't locate the relevant code, say so in the report ("could not locate webhook handler — skipped").
- **Don't run all 14 categories on a tiny app.** If the stack detection shows no Stripe, no DB, no uploads — skip those sections rather than pad the report with N/A entries.
- **Don't echo back the entire catalog as the report.** Only findings + skipped sections.
- **Don't claim "no issues found" without naming what you actually inspected.** If the report has zero High+ findings, list the file paths and grep patterns that produced that conclusion.
