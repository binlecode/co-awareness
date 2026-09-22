---
name: publish-cover
description: Publish or redeploy the marketing/design cover page (docs/cover.html) to Cloudflare Pages as <project>.pages.dev. Use when asked to publish, redeploy, or update the cover / landing / web page, or after cutting a release when the cover's version badge or embedded GIFs changed. Builds + audits the deploy bundle and deploys it, all automatically — wrangler runs on stored OAuth creds, but every wrangler line needs `env -u CLOUDFLARE_API_TOKEN` (see §2). Only a fresh login is a human step.
---

# Publish the web cover

Publishes `docs/cover.html` (the marketing/design cover page) to the public web as a **curated
subset** of the repo — just the cover page and the GIFs it references, served as its own clean site
root. Deployed to **Cloudflare Pages** as `<project>.pages.dev` via `wrangler` **direct upload** (we
hand Cloudflare a built folder — no repo access, no build step of its own).

Live URL: `https://co-awareness.pages.dev`. Scripts live in `scripts/` next to this file.

## What I can and can't do

- **Build + audit the bundle: automated** — run `scripts/build-cover-dist.sh` (below). Safe, deterministic.
- **Deploy: automated too, as long as auth is already on disk** — stored OAuth creds (§2) deploy fine
  non-interactively. Only a *fresh login* is a human step: if §2's `whoami` shows no valid creds,
  build + audit, then hand the user the login command to run via `!` — never attempt the login itself.

## 0. Decision gate (read first)

- **Curated subset, not the repo.** The site is the *built bundle* (`cover.html` → `index.html` + only
  the GIFs it references), not the repo's files at repo-shaped URLs.
- **Third-party art goes public.** The cover embeds character GIFs (Ghibli / Pinterest). Displaying
  them is the owner's call — a *content* decision, not a technical blocker. Already accepted on prior
  publishes; only re-flag if the art set changed.
- **Cloudflare account exists.** Free tier is enough.

## 1. Build + audit the bundle (automated)

```bash
.claude/skills/publish-cover/scripts/build-cover-dist.sh
```

Assembles `tmp/cover-dist/` and self-scores: path rewrite (`../gifs/`→`gifs/`), no secrets/local
paths, self-contained assets (no broken `src`), outbound links (expect only the repo URL), and prints
the version badge. Exits non-zero on any failure. Optionally eyeball it: `open tmp/cover-dist/index.html`.

**Version badge:** `docs/cover.html` carries a footer badge (`vX.Y.Z`) that is **not** covered by
the QA version-parity check, so it drifts silently (it has lagged a release before). Bump it in
`docs/cover.html` when you cut a release, rebuild, and redeploy so the public page matches the shipped
version. The script prints the badge it bundled — confirm it matches.

## 2. Deploy

Run wrangler via `npx` (no global install). Direct upload grants Cloudflare no repo access.

🔴 **Drop `CLOUDFLARE_API_TOKEN` from the environment — `env -u` on *every* wrangler line.** This repo
deploys on **OAuth creds stored on disk** (`~/Library/Preferences/.wrangler/config/default.toml`),
which work non-interactively and need no login. But this machine also exports an unrelated
`CLOUDFLARE_API_TOKEN` from `~/.zshrc`, wrangler **prefers the env token over stored OAuth**, and that
token has no Pages permission — so leaving it set fails with `Authentication error [code: 10000]`
followed by `Failed to automatically retrieve account IDs`. That error means *wrong credential picked*,
not *not logged in*; `wrangler login` is the wrong reflex and now refuses outright ("You are logged in
with an API Token. Unset the CLOUDFLARE_API_TOKEN..."). Confirm the right creds first:

```bash
env -u CLOUDFLARE_API_TOKEN npx wrangler whoami    # expect: "logged in with an OAuth Token" + an account row
env -u CLOUDFLARE_API_TOKEN npx wrangler pages deploy tmp/cover-dist --project-name=co-awareness --commit-dirty=true
```

Only if `whoami` shows no OAuth creds is a login needed — that one is the human's (`env -u
CLOUDFLARE_API_TOKEN npx -y wrangler login`, browser OAuth). Fixing the env token instead (grant it
Account → Cloudflare Pages → Edit, plus `CLOUDFLARE_ACCOUNT_ID`, since it can't enumerate accounts) is
a valid alternative, but it's the owner's call — that token is scoped for something else.

First run offers to create the project; accept. Re-running the same command redeploys to the same URL.
The `pages.dev` namespace is global across all accounts — keep the distinctive `co-awareness`
project name to avoid collisions.

## 3. Verify (live)

```bash
URL=https://co-awareness.pages.dev
curl -sSI "$URL" | head -1                                   # expect: HTTP/2 200
# Badge, retried: the edge can serve the PREVIOUS deploy for a few seconds after a successful
# upload, so one fetch showing the old version is not a failed deploy (see below).
for i in 1 2 3; do curl -sS "$URL" | grep -oE 'class="badge">v[0-9.]+'; sleep 3; done
grep -oE 'gifs/[A-Za-z0-9._-]+\.gif' tmp/cover-dist/index.html | sort -u \
  | while read -r g; do printf '%s ' "$g"; curl -sSo /dev/null -w '%{http_code}\n' "$URL/$g"; done
open "$URL"                                                  # eyeball layout + GIF smoothness
```

All `200`s and a page matching the local bundle = done.

**A single stale fetch is not a failed deploy — don't redeploy on it.** The edge can keep serving the
previous deploy for a few seconds after wrangler reports `Deployment complete`, so one fetch showing
the old badge proves nothing. This is the **mirror** of the failure §3 exists to catch: the check is
here to stop a stale page passing as current, so a false alarm the other way is what tempts someone
to stop trusting it. Settle which case you have before touching anything:

```bash
curl -sS "https://<deployment-hash>.co-awareness.pages.dev" | grep -oE 'class="badge">v[0-9.]+'
curl -sS -H 'Cache-Control: no-cache' "$URL?cb=$RANDOM"        | grep -oE 'class="badge">v[0-9.]+'
```

The deployment-specific URL wrangler prints is served straight from that upload, so it proves what
was actually shipped. New badge there + old badge on production = cache, wait it out. **Old badge on
the deployment URL = a genuinely bad bundle** — rebuild (§1), don't just redeploy.

**Don't ask `wrangler pages project list` whether a deploy went to production** — it reports the
production column as `No` even for a deploy that *is* Production. `wrangler pages deployment list` is
the command that answers it: it prints Environment, Branch, and the source commit per deployment.

## Notes

- **Snapshot, not live.** Editing `docs/cover.html` or a GIF does nothing to the live page until you
  rebuild (§1) + redeploy (§2). This is deliberate — the cover is a hand-curated snapshot, not a doc
  site that redeploys on every commit. (Hence direct upload over a GitHub-Pages / CF-Git integration.)
- **Independent surfaces.** The app reads the on-disk `gifs/`, and a Claude-hosted artifact bakes GIFs
  in as base64 — both are independent of this Pages site. Updating one does not update the others.
- **Assets:** see the `build-visuals` skill for (re)building the GIFs the cover embeds.
- **Teardown:** `npx wrangler pages project delete co-awareness` (removes the site + URL);
  `rm -rf tmp/cover-dist`.
- **Reuse for other repos:** same flow — build a `tmp/cover-dist/`, deploy with a distinct
  `--project-name`. Identical whether the repo is public or private (direct upload sends only the folder).

## Scratch files

The bundle lives in `tmp/cover-dist/` (repo root) — throwaway, never committed.
