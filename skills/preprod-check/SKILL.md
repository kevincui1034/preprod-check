---
name: preprod-check
description: >-
  Use this skill when the user asks for a pre-production readiness check,
  launch checklist, "is this safe to ship", "go-live review", "prod audit",
  or any variant of "what should I check before deploying". Fans a structured
  audit out across parallel sub-agents covering auth/multi-tenancy, input
  validation, billing & credit integrity, rate limiting, cost containment,
  external-request safety (SSRF, uploads), secrets/env, security
  headers/cookies, error handling, CORS, database, logging & monitoring,
  email/password flows, AI/LLM safety, performance & scalability,
  legal/compliance, operations, and release safety & operability
  (feature flags, rollback, backups, runbook). Adapts checks to the detected stack
  (Next.js, Auth.js, Stripe, Supabase, Drizzle, AI SDKs, blob storage, etc.),
  verifies findings before reporting, and produces a severity-grouped report
  with drafted patches for trivial fixes. Also invocable as `/preprod-check`.
metadata:
  author: kevincui1034
  version: "1.2.0"
---

# 🚀 Pre-Prod Readiness Check

Audit a project for production readiness across **17 categories**, then produce a **severity-grouped** report with `file:line` refs and drafted patches for the trivial fixes.

---

## At a glance

| | |
|---|---|
| **What it does** | Greps the codebase for known launch-blocking gaps — auth bypass, money leaks, SSRF, secret exposure, missing rate limits, and more. |
| **How it runs** | Detects the stack → **fans the audit out across parallel sub-agents** (one per category cluster) → merges + **verifies every Critical/High** → reports. |
| **What you get** | A severity-ordered findings list (Critical → Low), each one line with a `file:line` and a fix sketch, plus drafted patches for the mechanical fixes on request. |
| **Default stack** | Assumes a TypeScript/Next.js web app but degrades gracefully — categories that don't apply to the detected stack are skipped, not padded. |

> **Read-only until you approve.** The audit phase only inspects. Patches are drafted one-at-a-time in the final step and never written without explicit approval.

---

## Category map

Fan the audit out along these clusters. Each is one sub-agent; skip any whose categories are all N/A for the stack.

| Cluster | Categories | Typical ceiling |
|---|---|---|
| 🔐 **Access & tenancy** | 1 Auth & multi-tenant · 2 Input validation & injection | Critical |
| 💳 **Money & abuse** | 3 Billing & credit integrity · 4 Rate limiting · 5 Cost containment | Critical |
| 🛡️ **Perimeter & secrets** | 6 External-request safety (SSRF/uploads) · 7 Secrets & env · 8 Headers & cookies | Critical |
| 🩺 **Reliability & data** | 9 Error handling · 10 Database · 11 Logging & monitoring | High |
| 👤 **Accounts, legal & ops** | 12 Email & password · 13 Legal & compliance · 14 Operations & supply chain · 17 Release safety & operability | High |
| 🤖 **AI/LLM safety** | 15 AI/LLM safety *(only if an AI SDK is detected)* | High |
| ⚡ **Performance** | 16 Performance & scalability | High |

---

## Workflow

Run the steps in order. Discovery (Step 1) feeds the fan-out (Step 2); don't skip it or the checks go generic.

### Step 1 — Stack detection (parallel reads)

Read these in parallel, then write a **one-paragraph stack summary** before any checks:

| Read | For |
|---|---|
| `package.json` | frameworks, AI SDKs, payment SDKs, ORM, auth libs |
| `next.config.*` / `vite.config.*` | framework config |
| `tsconfig.json` | strict mode, path aliases |
| `CLAUDE.md` / `AGENTS.md` | project conventions |
| `.env.example` | declared env surface |
| `src/lib/db/schema.*` / `prisma/schema.prisma` | DB shape |
| `src/proxy.ts` / `src/middleware.ts` / `src/auth.*` | auth wiring |
| `src/app/api/**` / `pages/api/**` (Glob, don't read all) | API surface |

**Detect and record:** framework + version · auth library · payment processor · DB + ORM · AI/external APIs · file storage · hosting target. This summary is passed verbatim to every sub-agent.

### Step 2 — Fan the audit out to sub-agents

The categories are independent, so audit them **concurrently**. Spawn one sub-agent per applicable cluster (see the Category map), in a **single batch** so they run in parallel. Give each agent: (a) the Step 1 stack summary, and (b) the catalog sections for its cluster.

**Rules for the fan-out** (these are what make parallel audits reliable):

- **Leaf agents only.** Each agent greps/reads and reports itself — it must **not** spawn its own sub-agents (nested agents stall and return status text instead of findings).
- **Read-only.** Audit agents inspect and report; they do **not** edit. Patching happens in Step 5, in the main loop, after approval.
- **Structured, evidence-bearing return.** Require each agent's final message to be a findings list — one row per finding:

  `severity | file:line | what's wrong | fix sketch | evidence`

  The **evidence** field (the grep hit / the actual line that proves it) is mandatory — it's what lets you verify without re-running the whole search.
- **Evidence-driven, not vibes.** An agent must not report "missing CSP" unless it actually grepped for `Content-Security-Policy` and confirmed absence. Tell it so.

**Then, back in the main loop:**

1. **Merge & de-dupe** across agents — the same gap (e.g. a missing header) can surface from two clusters.
2. **Verify every Critical and High before reporting.** Re-open each cited `file:line` and confirm the evidence really shows the problem — audit agents over-flag. Demote or drop anything you can't confirm. For a deep audit, spawn one adversarial verifier per Critical whose job is to *disprove* it; keep the finding only if it survives.
3. Assign final severity (Step 3) and write the report (Step 4).

> For a large or repeatable audit, the whole fan-out → verify → synthesize can run as a single deterministic `Workflow` — but only when the user has opted into orchestration. Otherwise plain parallel `Agent` calls are fine.

### Step 3 — Severity rubric

| Severity | Meaning | Examples |
|---|---|---|
| 🔴 **Critical** | Money loss, data leak, or auth bypass possible **today** | Webhook skips idempotency insert; credit ledger has no concurrency guard; server action missing `auth()`; SSRF on a URL fetcher; secret in client bundle |
| 🟠 **High** | Realistic incident path; not actively bleeding | No per-user $ ceiling; no CSP; upload mime not sniffed; reset token never expires; no env validation at boot |
| 🟡 **Medium** | Hygiene gap; would slow incident response or DX | No structured logs; no Sentry filter; FK column lacks index; no health check |
| ⚪ **Low** | Polish; embarrassing but not damaging | 500 page leaks stack trace; missing privacy-policy link; cookie flags left to framework defaults |

When in doubt, **err one level higher** — a false-positive Critical is cheaper than a missed one. (This is why Step 2 verifies Critical/High: err high while auditing, then confirm before it reaches the report.)

### Step 4 — Output the report

Group by **severity**, not by category. One line per finding.

```
## Pre-Prod Findings — <project name> (<date>)

**Stack detected**: <one-line summary>
**Categories checked**: <list, with skipped ones in parens>

### 🔴 Critical (N)
- [file.ts:42](src/file.ts#L42) — <what's wrong>. Fix: <one-line sketch>.

### 🟠 High (N)
- ...

### 🟡 Medium (N)
- ...

### ⚪ Low (N)
- ...

### Skipped / not applicable
- <category>: <reason>

### Patches I can draft now
- <finding> — would touch <file:line>
```

Keep each finding to **one line** — detail belongs in the patch, not the report.

### Step 5 — Propose patches

After the report, ask which findings to patch. For each approved one:

- Show a unified diff (or a full file for new files).
- Stick to the mechanical fixes in **Patch templates**. Don't draft business-logic changes (e.g. a credit-ledger redesign) — flag those as needing a follow-up plan.
- **One finding, one patch, one approval.** Always wait for explicit approval before editing; don't batch.

---

## Check catalog

The source of truth for what to inspect. Each row: **the check** · **how to detect it** · **default severity**. Refer to it during Step 2; report using the severity rubric, not by category.

### 1. Authentication & multi-tenant isolation

| Check | How to detect | Sev |
|---|---|---|
| Every server action / route handler starts with an auth check | grep `actions.ts` + `server-only` modules for `await auth()` (or equiv); flag any handler missing it | 🔴 |
| Cross-tenant joins re-verify ownership server-side | trace request-supplied FK ids (`userId`, `projectId`, …) — must re-scope to `session.user.id` before use | 🔴 |
| Session/JWT callback refreshes sensitive fields per-request | if plan / ban / session-invalidation is mirrored on the session, the callback must re-read it | 🟠 |
| Middleware/proxy **and** layout both gate auth (defense-in-depth) | single layer only | 🟡 |
| Password reset bumps a session-invalidation timestamp | revokes other sessions; missing | 🟠 |

### 2. Input validation & injection

| Check | How to detect | Sev |
|---|---|---|
| Parameterized queries everywhere | grep template-string SQL `` `… ${…}` `` outside `sql` tagged templates | 🔴 |
| Zod (or similar) on every server-action input | grep `formData.get(` flowing to a DB write with no schema parse | 🟠 |
| No `dangerouslySetInnerHTML` on user content without sanitization | grep the sink; require DOMPurify or equiv | 🟠 |
| Path-traversal guard on file ops | `fs.*`/blob keys from user input must `path.normalize` + reject `..` | 🟠 |
| Redirect-URL validation | open redirects via `?next=`/`?callbackUrl=` must allowlist host | 🟠 |

### 3. Billing & credit integrity *(skip if no payments / no in-app currency)*

| Check | How to detect | Sev |
|---|---|---|
| Webhook signature verified before any DB write | grep route for `constructEvent` (or equiv) ahead of body parse | 🔴 |
| Webhook idempotency — insert event id first, bail on conflict | grep handlers for the idempotency insert | 🔴 |
| Credit-ledger concurrency guard | `SELECT … FOR UPDATE`, version column, or `CHECK (balance >= 0)` | 🔴 |
| Refund on every failure branch of a charged external call | trace each error path; list any that don't refund | 🔴 |
| Monthly grant uses `GREATEST(balance, allotment)` (not blind set) | grep the grant | 🟠 |
| Cost / price tables stay in sync | if credit-cost and USD-per-unit live in separate files, flag tuples present in one, missing in the other | 🟠 |

### 4. Rate limiting & abuse

| Check | How to detect | Sev |
|---|---|---|
| Per-user daily **$ ceiling** on AI/render spend (independent of quota) | catches runaway loops / prompt-injection cost attacks | 🟠 |
| Quota/credit gate on every user-triggered external call | grep `fetch(` to AI/scraper hosts in server modules; verify a gate precedes each | 🟠 |
| IP rate limit on signup / login / password reset | Upstash, Vercel KV, or similar on auth endpoints | 🟠 |
| Webhooks reject unsigned requests fast (cheap 401 before work) | grep for early signature check | 🟡 |

### 5. Cost containment & external API safety

| Check | How to detect | Sev |
|---|---|---|
| Timeout on every external call | grep `fetch(` without `AbortSignal.timeout`; must be < platform function ceiling | 🟠 |
| Cost-anomaly alert (> N× rolling-median daily spend) | some Sentry/PostHog/cron mechanism | 🟡 |
| Circuit breaker / retry budget on flaky providers | missing means a provider outage hangs functions | 🟡 |
| `maxDuration` / `memory` matches actual work | mismatch → silent timeouts | 🟡 |

### 6. External-request safety (SSRF, uploads, blob URLs)

| Check | How to detect | Sev |
|---|---|---|
| SSRF blocklist on any server fetch of a user-supplied URL | resolve DNS first, then check against the blocked ranges below | 🔴 |
| Upload mime **sniffed** server-side (first bytes, not `Content-Type`) | grep upload route | 🟠 |
| Size caps enforced server-side (not just client) | grep `maxSize`/content-length on upload routes | 🟠 |
| Blob/storage URL host validation | pin URLs returned from the upload service to your bucket/blob host | 🟠 |

> **SSRF blocked ranges:** `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `169.254.0.0/16` (**cloud metadata!**), `::1`, `fc00::/7`. Resolve the hostname to an IP first, then test the IP — a public hostname can resolve to a private address.

### 7. Secrets & env

| Check | How to detect | Sev |
|---|---|---|
| No secrets in the client bundle | grep `NEXT_PUBLIC_` (or equiv) for anything key/secret/token-shaped | 🔴 |
| `.env.local` not tracked in git | `git check-ignore .env.local`; `.env.example` is the only committed env file | 🔴 |
| Env validation at boot (schema throws on missing required vars) | grep for a zod/t3-env schema imported by the entrypoint | 🟠 |
| Secrets not logged | grep `console.log` near auth/payment/AI modules for env-named vars | 🟠 |

### 8. Security headers & cookies

| Check | How to detect | Sev |
|---|---|---|
| CSP set (via middleware/proxy or `headers()`) | grep `Content-Security-Policy` | 🟠 |
| HSTS, `X-Frame-Options`, `Referrer-Policy`, `X-Content-Type-Options: nosniff` | grep each | 🟡 |
| Session cookie flags: `httpOnly` + `secure` + `sameSite` | verify explicit, not relying on framework defaults | 🟡 |
| CORS allowlist explicit (not `*`) on authed endpoints | grep for wildcard origin | 🟠 |

### 9. Error handling & UX fallbacks

| Check | How to detect | Sev |
|---|---|---|
| No stack traces / raw messages in user-facing responses | grep API catches returning `error.stack`/`error.message` | 🟠 |
| Error boundaries on major route segments (`error.tsx` or equiv) | grep the segments | 🟡 |
| Friendly fallback for quota/rate-limit/credit-exhausted (expected, not exceptional) | grep how those errors surface | 🟡 |
| `not-found` / 404 page exists | grep | ⚪ |

### 10. Database

| Check | How to detect | Sev |
|---|---|---|
| Indexes on hot columns (FKs in joins/filters, sorted `created_at`, `WHERE` cols) | read schema; grep `.where(`/`.eq(` in top modules | 🟡 |
| `ON DELETE` semantics right on sensitive tables (`CASCADE` owned, `RESTRICT` audit) | read schema | 🟡 |
| Backups / PITR enabled on prod DB | can't verify in code — surface as a manual check | 🟡 |
| No `db:push` against prod (dev-only) | grep scripts / docs for misuse | ⚪ |

### 11. Logging & monitoring

| Check | How to detect | Sev |
|---|---|---|
| No PII in logs (email, phone, names, tokens) | grep logging calls | 🟠 |
| Structured (JSON) logs in prod, not bare `console.log` strings | grep logger usage | 🟡 |
| Error tracker (Sentry/equiv) with `beforeSend` dropping expected user errors | grep init | 🟡 |
| Health-check endpoint returns DB + critical-dep status | grep `/api/health` | 🟡 |

### 12. Email & password

| Check | How to detect | Sev |
|---|---|---|
| Password hashing is argon2id / bcrypt with sane cost | grep hashing; plain SHA/MD5/unhashed | 🔴 |
| Reset tokens single-use, expiry ≤ 1h, signed link | grep token issue/verify | 🟠 |
| SPF / DKIM / DMARC on sender domain | can't verify in code — surface as manual DNS check | 🟠 |
| Email verification required before account-affecting actions | grep the flows | 🟡 |
| Bounce/complaint handling on the email provider | grep webhook/config | 🟡 |

### 13. Legal & compliance

| Check | How to detect | Sev |
|---|---|---|
| Privacy policy + ToS linked from signup + footer (Stripe/OAuth require it) | grep routes/footer | 🟠/⚪ |
| Account deletion that purges/anonymizes PII + cascades FKs (GDPR/CCPA) | grep for the path | 🟡 |
| Cookie consent if serving EU traffic with non-essential cookies | grep | 🟡 |
| Data-export endpoint | grep | ⚪ |

### 14. Operations & supply chain

| Check | How to detect | Sev |
|---|---|---|
| `npm audit` / `pnpm audit --prod` — report HIGH/CRITICAL runtime CVEs | run it | 🟠 |
| CI/CD secrets not echoed in logs | spot-check `.github/workflows/` for `echo $SECRET`-style | 🟠 |
| Lockfile committed; no `^` ranges on security-critical deploy deps | check lockfile presence | ⚪ |
| Staging/preview env separate from prod | one-env setup | 🟡 |
| No `--no-verify` / `--no-gpg-sign` in committed scripts | grep | ⚪ |

### 15. AI / LLM safety *(skip if no AI/LLM SDK)*

| Check | How to detect | Sev |
|---|---|---|
| Prompt-injection trust boundary on tool-calling agents | trace untrusted content (scraped pages, uploads, transcripts) into a model that can call privileged tools | 🟠 |
| Model output not rendered as raw HTML without sanitization (stored XSS via `<script>`/`javascript:`) | grep `dangerouslySetInnerHTML` / markdown renderers fed model output | 🟠 |
| Per-request token/cost cap (max output tokens set, input truncated) | grep model calls; unbounded context (long transcripts, recursive loops) = runaway spend | 🟠 |
| Tool-calling / agent loops have a max-iteration cap | grep the loop | 🟡 |
| PII/secrets not sent to third-party models beyond need; a data stance exists | trace user data into external-provider prompts | 🟡 |
| System prompt not leaked to client (bundle or responses) | grep client-reachable code | 🟡 |

### 16. Performance & scalability

| Check | How to detect | Sev |
|---|---|---|
| No unbounded queries — list endpoints paginate / `LIMIT` | grep `.findMany(`/`select` without a limit on list routes | 🟠 |
| No N+1 in hot paths (query-in-a-loop) | grep `await` inside `.map(`/`for` over DB calls | 🟡 |
| Connection pooling correct for serverless (pgbouncer / driver mode) | check DB URL + driver against hosting model | 🟠 |
| Caching on expensive read-hot endpoints (or a documented reason not to) | grep for cache usage / `revalidate` | 🟡 |
| Large payloads streamed, not buffered wholly in memory | grep routes returning big blobs/arrays | 🟡 |

### 17. Release safety & operability

Can you survive a bad deploy? These are as much process as code — surface the ones code can't confirm as **manual checks** in the report, but grep for the signals below.

| Check | How to detect | Sev |
|---|---|---|
| Feature flag / kill-switch to disable a risky feature in prod **without a redeploy** | grep for a flag lib (LaunchDarkly, Statsig, Unleash, PostHog flags) or env/DB-backed toggles gating payment/AI/external-facing features; none = a bad feature can only be turned off by shipping | 🟡 |
| Deploy is **rollback-safe** — the previous release can be redeployed cleanly | mostly process (manual check), but flag code that hard-breaks old clients: a removed/renamed API field still read by shipped clients, no API versioning, a breaking response-shape change | 🟠 |
| Latest migration is **reversible / non-destructive** | read the newest migration for `DROP COLUMN` / `DROP TABLE` / type-narrowing with no down migration — an irreversible migration blocks the rollback above | 🟠 |
| Backup **restore actually rehearsed** (not just backups enabled) | can't verify in code — surface as a manual check; §10 covers that backups *exist*, this covers that a restore has been *tested* (an untested backup is not a backup) | 🟠 |
| On-call **runbook / incident playbook** exists and is linked | grep for `RUNBOOK.md` / incident docs; confirm it's referenced from the README or ops docs, not just sitting unlinked | 🟡 |

---

## Patch templates

The trivial, known-shape fixes the skill may draft (mechanical, not architectural).

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
      { key: "Content-Security-Policy", value: "<draft a CSP from the detected stack — be conservative; flag inline scripts in the report>" },
    ],
  }];
}
```

For CSP, **don't draft a wildcard `unsafe-inline` policy** to make it "work" — list the actual hosts the app loads from (payment SDK, fonts, analytics) based on grep findings.

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

### Webhook idempotency (Drizzle example)

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

Call `await assertPublicHost(url)` before any server-side fetch of a user-supplied URL.

### Upload mime sniffing

```ts
import { fileTypeFromBuffer } from "file-type";

const head = Buffer.from(await blob.slice(0, 4100).arrayBuffer());
const sniffed = await fileTypeFromBuffer(head);
if (!sniffed || !ALLOWED_MIMES.has(sniffed.mime)) {
  return new Response("Unsupported file type", { status: 415 });
}
```

### Cookie flags (Auth.js v5)

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

- **Don't draft architectural patches.** A credit-ledger race fix is a design discussion, not a Write call. Flag it and stop.
- **Don't let audit sub-agents edit or spawn their own sub-agents.** They are read-only leaves that return findings; patching and orchestration stay in the main loop.
- **Don't report a Critical/High you haven't verified.** Agents over-flag — re-open the cited line and confirm the evidence before it reaches the report.
- **Don't run destructive or money-spending commands.** No `db:push`, no paid API calls, no installing new deps without approval.
- **Don't invent findings.** If you couldn't locate the relevant code, say so ("could not locate webhook handler — skipped").
- **Don't run all 17 categories on a tiny app.** Skip inapplicable clusters rather than pad the report with N/A entries.
- **Don't echo the whole catalog back as the report.** Only findings + skipped sections.
- **Don't claim "no issues found" without naming what you inspected.** Zero High+ findings must be backed by the file paths and grep patterns that produced that conclusion.
