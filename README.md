# RevOps PAI App Starter

Template for building and shipping **small, single-purpose web apps** with PAI / Claude Code — clone, build, deploy to Vercel in minutes.

This is a GitHub **template repository**. It's the home for the kind of quick, focused apps RevOps spins up with AI: a calendar over a Google Sheet, a dashboard, an internal tracker, a one-page tool. Each app you make starts as its own clean repo from this template — *not* a shared folder everyone commits into.

## Who it's for

Circle RevOps and anyone building with PAI who wants a consistent, deploy-ready starting point instead of re-deciding conventions (and re-learning the same Vercel gotchas) every time.

## Use it

1. Click [Use this template](https://github.com/circleco/revops-pai-app-starter#) → Create a new repository (gives you a clean repo, fresh history — no fork link, no secrets, no deploy hooks carried over).

   ![Use this template](./docs/use_template.png)

2. Clone your new repo and build your app. Keep it self-contained: ideally a single static `index.html` (or a small set of files) plus `vercel.json`.
3. Let Claude Code drive — it reads `CLAUDE.md` and follows the conventions automatically.

## Deploy (Vercel)

```bash
bunx vercel --prod        # first run links the project, then deploys
```

- Serve your app at `/` — see the `cleanUrls` + root-rewrite note in `CLAUDE.md` (a common 404 trap).
- **Access reality check:** a Vercel password or SSO gate protects the *page*, not data your app fetches **client-side** (e.g. a Google Sheet read in the browser). For anything sensitive, restrict the **data source** itself — don't rely on a front-end gate. See `CLAUDE.md` → *Security*.
- If your Vercel team has Deployment Protection on by default, fresh deploys may **401** until you adjust it. `CLAUDE.md` covers both opening it up and keeping it gated.

## Conventions (the short version)

- **bun / bunx** always — never npm / npx.
- **TypeScript** over Python for any new scripting.
- **Static-first** — prefer a no-build static app; reach for a framework only when you actually need one.
- **No secrets in the repo.** Scan before you commit. Use `.vercelignore` to keep non-app files out of the deploy and `.gitignore` for `.vercel` / `node_modules`.

**Full conventions, deploy recipe, and the hard-won gotchas live in [`CLAUDE.md`](./CLAUDE.md).** Read it first.
