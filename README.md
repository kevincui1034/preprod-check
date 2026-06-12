# Pre-Prod Readiness Check

A [Claude Code](https://code.claude.com/docs) skill that audits a project for
production readiness across **14 categories** and produces a severity-grouped
findings report with drafted patches for trivial fixes.

It adapts to your detected stack (Next.js, Auth.js, Stripe, Supabase, Drizzle,
AI SDKs, blob storage, etc.) and skips categories that don't apply.

## What it checks

Auth & multi-tenancy · input validation · billing & credit integrity · rate
limiting · cost containment · external-request safety (SSRF, uploads) ·
secrets / env · security headers & cookies · error handling · CORS ·
database (indexes, backups) · logging & monitoring · email / password flows ·
legal / compliance · operations.

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
pins `version: 1.0.0`, you'll receive changes when that version is bumped.

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
