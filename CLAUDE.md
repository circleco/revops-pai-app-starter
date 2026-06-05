# CLAUDE.md — RevOps PAI App Starter

Instructions for Claude Code / PAI working in any app created from this template.

## What this repo is

A **single small, single-purpose web app**, deployed to Vercel. Created from the `revops-pai-app-starter` template, so it's its own clean repo — one app per repo, not a monorepo. Typical shape: a static `index.html` (often reading a Google Sheet or an internal API in the browser) plus `vercel.json`. Keep it that simple unless the task genuinely needs more.

## Setup: zero to deployed (the agent runs this)

Get a teammate from nothing to a live app with the fewest manual steps. Run these checks in order. Only stop for the three steps a human genuinely must do — marked 🧑 (account creation + the two interactive logins). Everything else, run unattended.

### 1. GitHub CLI + auth

```bash
gh --version || brew install gh          # macOS; otherwise https://cli.github.com
gh auth status || gh auth login          # 🧑 interactive — user completes the browser/device flow
gh auth setup-git                        # wire git to use gh credentials (idempotent)
```

- **No GitHub account yet?** 🧑 There is no CLI or API to create one — send the user to <https://github.com/signup>, then run `gh auth login`.

### 2. Create the app repo from this template

```bash
gh repo create circleco/<app-name> \
  --template circleco/revops-pai-app-starter \
  --private --clone
cd <app-name>
```

- Template source: <https://github.com/circleco/revops-pai-app-starter>.
- `--private` by default (internal tools); `--public` only if the app is meant to be.
- Creating under `circleco` needs org membership + repo-create permission. **No org access?** Create under the user's own account, then grant Rob admin so he can manage/transfer it:
  ```bash
  gh repo create <app-name> --template circleco/revops-pai-app-starter --private --clone
  cd <app-name>
  # share with Rob as admin (run from inside the repo; {owner}/{repo} auto-resolve):
  gh api --method PUT "repos/{owner}/{repo}/collaborators/robertowtr-circle" -f permission=admin
  ```
  GitHub collaborator invites are **by username, not email** — use `robertowtr-circle` (Rob's GitHub login), not `roberto.walter@circle.co`. Rob receives an invitation and must accept it; once accepted he has admin and can transfer the repo into the org.
- `--clone` leaves you in the working copy. Add `--include-all-branches` only if you actually need the template's non-default branches.

### 3. Vercel CLI + auth

```bash
bunx vercel --version || bun add -g vercel
bunx vercel whoami || bunx vercel login  # 🧑 interactive — user completes login
```

### 4. Link, configure, deploy

```bash
bunx vercel link --yes                          # link local dir to a Vercel project
echo "$SHEET_ID" | bunx vercel env add SHEET_ID production   # pipe the value so it's non-interactive
bunx vercel env pull .env.local                 # gitignored local copy for dev
bunx vercel --prod                              # deploy
```

All config goes through env vars (see **Security standards** — yes, including Sheet IDs). Then verify the live URL (see **Verify before claiming done**) and adjust Deployment Protection if the team default isn't what you want (gotcha #2).

### What still needs a human (don't pretend otherwise)

- 🧑 **Creating a GitHub account** — web signup only, no automation path.
- 🧑 **`gh auth login` and `vercel login`** — interactive auth. The agent runs the command; the human completes it. Both *can* run non-interactively if `GH_TOKEN` / `VERCEL_TOKEN` are present in the environment — read them from env, never hardcode a token.

## Conventions

- **bun / bunx always — never npm / npx.** Deploy with `bunx vercel --prod`.
- **TypeScript over Python** for any new script. Don't add Python unless asked.
- **Static-first.** Prefer a no-build static app. Add a framework/build step only when the app actually requires it — most RevOps tools don't.
- **Markdown for docs**, not HTML.

## Deploy recipe (Vercel)

```bash
bunx vercel --prod        # first run links/creates the project, then deploys
```

Serve the app at `/`. Two ways:
- **Simplest:** name the entry file `index.html` — Vercel serves it at `/` with zero config.
- **Or** keep a descriptive filename and add a `vercel.json` rewrite (see gotcha #1 below — get the path right).

## Gotchas — read before deploying (these have all bitten us)

1. **`cleanUrls` strips `.html`, which breaks a naive root rewrite.**
   With `"cleanUrls": true`, `/my-app.html` 308-redirects to `/my-app`. So a rewrite of `/` → `/my-app.html` 404s. Point the rewrite at the **clean** path:
   ```json
   { "cleanUrls": true, "rewrites": [{ "source": "/", "destination": "/my-app" }] }
   ```
   Or just use `index.html` and skip the rewrite entirely.

2. **Your Vercel team may have Deployment Protection (SSO) ON by default → fresh deploys 401.**
   Symptom: `curl -I <url>` returns `401` with a `_vercel_sso_nonce` cookie. That's Vercel Authentication, not your app.
   - To **keep it gated to the team**: leave it on (free; gates to Vercel team members).
   - To **open it** (link-accessible to anyone): Project → Settings → Deployment Protection → turn off Vercel Authentication, or via API:
     ```bash
     curl -X PATCH -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
       "https://api.vercel.com/v9/projects/$PROJECT_ID?teamId=$TEAM_ID" -d '{"ssoProtection": null}'
     ```
     (Token from `~/Library/Application Support/com.vercel.cli/auth.json`; IDs from `.vercel/project.json`. Keep the token in the header, never in a URL or a log.)
   - **Password Protection** (custom shared password) is a **Pro-plan feature** — unavailable on Hobby.

3. **A front-end gate does NOT protect client-side-fetched data.** *(The important one.)*
   If the app reads a Google Sheet (gviz/JSONP) or any source **in the browser**, that source must be reachable by the browser — i.e. effectively public. A Vercel password/SSO gate then protects only the **HTML shell**: anyone past it (or anyone reading devtools/the network tab) can hit the data URL directly, forever, and the gate's password rotation does nothing to revoke it.
   - For **low-stakes** data: a gate is fine as *friction/discoverability reduction* — just don't call it security.
   - For **sensitive** data: move the boundary to the **data layer** — restrict the sheet/source itself (org-restricted Google Sheet + Google Site, or a serverless function reading with a service account so creds never reach the browser). Decide this *before* deploying.

## Security standards

These are non-negotiable for every app built from this template.

### Secrets — and config — live in environment variables, never hardcoded

**Never commit a secret.** No API keys, tokens, passwords, service-account JSON, or `.env*` files in the repo or in client-side code. Ever. If you're about to paste a key into a file, stop — it goes in an env var.

**Treat _all_ external config and identifiers as env vars too — including "public" ones.** Google Sheet IDs, spreadsheet GIDs, endpoint URLs, project IDs: none of them get hardcoded in committed source. Even when an ID is harmless to expose, route it through an env var, because (a) it removes the per-value judgment call about what's "sensitive enough" — exactly the call people get wrong, (b) it keeps your data sources out of the public repo, and (c) it makes the same app configurable per environment with zero code edits. **One rule, no exceptions: config goes in env vars.**

Install the Vercel CLI locally and manage secrets through it (bun/bunx, never npm):

```bash
bun add -g vercel                          # or use `bunx vercel` per-command
bunx vercel link                           # link this local dir to the Vercel project
bunx vercel env add MY_SECRET production   # add a secret (also: preview | development)
bunx vercel env pull .env.local            # pull into a gitignored local file for dev
```

Read secrets in code via `process.env.MY_SECRET` — **server-side / serverless only** (see the trap below). `.gitignore` **must** include `.env`, `.env.local`, `.env.*.local`, `.vercel`, `node_modules`, `.DS_Store`.

### The client-side secret trap (critical for static apps)

A purely static app (one `index.html`, no server) **cannot hold a secret**. Anything it reads at runtime, the browser — and anyone in devtools — reads too. Vercel env vars are available **only at build time and inside serverless/edge functions**, never to static client JS.

- If the app needs a **real secret** (private API key, write token), add a **Vercel serverless function** (`/api/*.ts`) that reads it from `process.env` server-side and returns only the data the client needs. The secret never ships to the browser.
- **Non-secret config still goes in an env var** (per the rule above) — the only open question is *how it reaches a static client*, since a no-build `index.html` can't read `process.env` at runtime. Two patterns:
  - **Serverless proxy (preferred when the value should stay off the client):** an `/api/*.ts` function reads the ID from `process.env` and does the fetch server-side; the ID never reaches the browser.
  - **Build-time injection (when the value is genuinely public, e.g. a link-shared Sheet ID):** a small deploy step writes the env var into a generated, gitignored `config.js` (or substitutes a placeholder in the HTML). The value still ends up visible in the shipped bundle — acceptable for a public ID — but it is **never committed to the repo** and stays per-environment.
- Practical consequence: once an app has *any* external config, even a "static" app carries a minimal build/inject step or a serverless function. That's the deliberate cost of the no-hardcoding rule.
- Same boundary as gotcha #3: protect data at its source, not with a front-end gate.

### Tokens & access

- **Never put a token in a URL** (query string or path) — it leaks to server logs, browser history, and referrer headers. Always use an `Authorization: Bearer <token>` header.
- **Least privilege:** scope every token and service account to read-only and to the single resource it needs. No broad/admin tokens in an app this small.
- **If a secret leaks, rotate it at the source immediately** — and re-key the data source (sheet/DB) if that's what was exposed. Rotating a front-end password does nothing.

### Before every commit

```bash
grep -rinE "(api[_-]?key|secret|token|password|bearer)\s*[:=]" --include="*.html" --include="*.json" --include="*.ts" .
```

Returns nothing → good. Returns a real value → remove it, move it to an env var, and (if it was ever committed) rotate it. Keep `.vercelignore` excluding non-app files (docs, ISAs, helper scripts, `.env*`) so they're never served from the deployment.

### App-level

- **Escape untrusted data before inserting it into the DOM** (HTML-escape) — anything from a sheet, an API, or user input — to prevent XSS.
- Commit your lockfile (`bun.lock`); keep dependencies minimal (every dep is attack surface).
- Don't log secrets — not to the console, not to error trackers.

## Verify before claiming done

- Deploy verified = the live URL probed, not "it pushed." `curl -I` for status, then **open it in a real browser** (PAI: Interceptor) to confirm it renders *and* that client-side data actually loads. A page that returns 200 can still be a blank grid if the data fetch fails — check.
- Add a visible failure state for client-side fetches: gviz silently breaks when a sheet tab/schema is renamed, leaving an always-live page blank with no error.

## Git

This is a Circle org repo — use branches/PRs, not direct pushes to `main`. Branch naming: `rw-YYYY-MM-DD-short-description`.
