# 🚀 Pre-Prod Readiness Check

A [Claude Code](https://code.claude.com/docs) skill that audits a project for
production readiness across **17 categories** and produces a severity-grouped
findings report with drafted patches for trivial fixes.

It **fans the audit out across parallel sub-agents** (one per category cluster),
merges and **verifies every Critical/High finding** before reporting, and adapts
to your detected stack (Next.js, Auth.js, Stripe, Supabase, Drizzle, AI SDKs,
blob storage, etc.) — skipping categories that don't apply.

## What it checks

Auth & multi-tenancy · input validation · billing & credit integrity · rate
limiting · cost containment · external-request safety (SSRF, uploads) ·
secrets / env · security headers & cookies · error handling · CORS ·
database (indexes, backups) · logging & monitoring · email / password flows ·
**AI / LLM safety** · **performance & scalability** · legal / compliance ·
operations · **release safety & operability** (feature flags, rollback,
tested restore, runbook).

## How it runs

1. **Detect** the stack (parallel reads of `package.json`, config, schema, auth wiring).
2. **Fan out** — one read-only sub-agent per category cluster, run concurrently, each returning evidence-bearing findings.
3. **Merge, de-dupe, and verify** — every Critical/High is re-checked against the cited `file:line` before it reaches the report (audit agents over-flag).
4. **Report** — severity-grouped, one line per finding, with a fix sketch.
5. **Patch** — drafts the mechanical fixes one at a time, only on your approval.

## What's new

### 1.2.0

- **New category — release safety & operability** (§17): feature flag /
  kill-switch to disable a feature without a redeploy, rollback-safe deploys,
  reversible/non-destructive migrations, a **rehearsed** backup restore (not
  just backups enabled), and a linked on-call runbook. Brings the total to
  **17 categories**.

### 1.1.0

- **Parallel sub-agent fan-out** — the 14→**16** categories are clustered into
  ~6 read-only leaf agents that audit concurrently, then the main loop merges,
  de-dupes, and **verifies every Critical/High** before reporting (audit agents
  over-flag; verification cuts false positives).
- **Two new categories** — **AI/LLM safety** (prompt-injection trust boundary,
  unsanitized model output, per-request token/cost caps, agent-loop caps) and
  **performance & scalability** (unbounded queries, N+1, serverless connection
  pooling, hot-path caching).
- **Readability overhaul** — an "at a glance" summary, a category-cluster map,
  and the whole check catalog reformatted from bullet-walls into scannable
  per-category tables with severity badges.

## Install

This repo is both a Claude Code **plugin** and a single-plugin **marketplace**,
so you can install it whichever way you prefer.

### Option A — Plugin (recommended)

In Claude Code, run:

```
/plugin marketplace add kevincui1034/preprod-check
/plugin install preprod-check@preprod-check
```

Update later with `/plugin marketplace update preprod-check`. Because the plugin
pins `version: 1.2.0`, you'll receive changes when that version is bumped.

### Option B — Drop the skill in manually

If you'd rather not use the plugin system, copy just the skill folder into your
skills directory:

**Personal (all projects):**

```bash
# macOS / Linux
git clone https://github.com/kevincui1034/preprod-check /tmp/preprod-check
cp -r /tmp/preprod-check/skills/preprod-check ~/.claude/skills/preprod-check
```

```powershell
# Windows (PowerShell)
git clone https://github.com/kevincui1034/preprod-check $env:TEMP\preprod-check
Copy-Item -Recurse "$env:TEMP\preprod-check\skills\preprod-check" "$env:USERPROFILE\.claude\skills\preprod-check"
```

**Project-scoped** (checked into a repo, shared with collaborators): copy the
same `skills/preprod-check` folder into your project's `.claude/skills/`.

## Usage

Once installed, invoke it explicitly:

```
/preprod-check
```

…or just ask in natural language — Claude triggers it on prompts like
"is this safe to ship?", "go-live review", "what should I check before
deploying?", or "prod audit".

## Repo layout

```
preprod-check/
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest
│   └── marketplace.json    # marketplace catalog (one plugin, source "./")
├── skills/
│   └── preprod-check/
│       └── SKILL.md         # the skill itself
├── LICENSE
└── README.md
```

## License

[MIT](LICENSE) © Kevin Cui
