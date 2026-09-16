---
description: Security-scan, push to a GitHub repo, deploy GitHub Pages via Actions, and update README + About section
argument-hint: <github-repo-url>
allowed-tools: Bash, Read, Edit, Write, Glob, Grep
---

## Context

- Target repo URL (may be empty if the user didn't pass one): $ARGUMENTS
- Current git status: !`git status 2>&1`
- Current remotes: !`git remote -v 2>&1`
- Files in the project root: !`ls -la`

## Goal

Publish the current project to GitHub end-to-end: scan for secrets first, then push, then stand up GitHub Pages via a GitHub Actions workflow, then write the README, then update the repo's About section (description + link) to point at the live Pages URL. Do all of this without ever uploading sensitive data.

Work through the phases below **in this order** — the security scan is a hard gate before anything touches the network. If the user gave a repo URL as an argument, use it; otherwise ask them for it before Phase 2 (don't guess a repo name).

## Phase 1 — Security scan (must pass before any push)

Scan the **entire working tree that would be committed** (staged + unstaged + untracked, respecting `.gitignore`) for anything that shouldn't leave this machine:

1. Secrets/credentials patterns: API keys, access/secret tokens, private keys (`BEGIN...PRIVATE KEY`), passwords, connection strings, JWTs, cloud credential blocks (AWS/GCP/Azure), `.env`/`.env.*` files, `*.pem`/`*.key`/`*.pfx`, `id_rsa*`, service-account JSON, `.npmrc`/`.pypirc` with tokens.
2. Personal/sensitive data that has no reason to be public (real emails/phone numbers/addresses hardcoded outside of clearly-fictional demo content, internal hostnames, internal ticket/incident references).
3. Anything that looks like a real company's proprietary data, logos, or trademarks if this is meant to be a public demo (cross-check against any project instructions, e.g. CLAUDE.md, about what is/isn't allowed).
4. Accidentally-large or binary files that don't belong in a static web project (build artifacts, node_modules, database dumps, `.zip`/`.tar` bundles).
5. Confirm `.gitignore` exists and actually excludes local/machine-specific files (e.g. `.claude/settings.local.json`, `.env`, OS cruft like `.DS_Store`/`Thumbs.db`).

Use `git status`/`git diff --cached`/`git diff` plus `Grep` across the tree for secret-shaped patterns (e.g. `(?i)(api[_-]?key|secret|token|password|BEGIN (RSA|EC|OPENSSH) PRIVATE KEY)`). Do not rely on filenames alone — check file contents.

**If anything suspicious is found**: stop, do not push, and report exactly what was found and where, letting the user decide (redact, `.gitignore` it, or explicitly approve). Never silently strip and continue. Only proceed to Phase 2 once the tree is clean or the user has explicitly approved specific findings as safe (e.g. deliberately fake demo data).

## Phase 2 — Push to the GitHub repo

1. If no git repo exists yet, `git init`.
2. If there's no commit author identity configured (`git config user.name` / `user.email` both locally and globally empty), ask the user what to use — never guess a name.
3. Determine the target remote URL: the one passed as `$ARGUMENTS`, or ask the user. Extract `owner/repo` from it for later API calls.
4. Add/update the `origin` remote to that URL (`git remote add origin <url>` or `git remote set-url origin <url>` if it already exists).
5. Stage and commit any pending changes with a concise message describing what's being published (skip if the tree is already committed and clean).
6. Ensure the default branch is `main` (`git branch -M main` if needed) and `git push -u origin main`.
7. Pushing over HTTPS may trigger a Git Credential Manager browser sign-in — if the push hangs, tell the user to check for a sign-in prompt rather than assuming failure.

## Phase 3 — GitHub Pages via GitHub Actions

1. Check for an existing Pages-deploy workflow under `.github/workflows/`. If one already exists and looks correct, reuse/update it rather than duplicating.
2. Otherwise create `.github/workflows/deploy-pages.yml` using the official Actions-based Pages flow (`actions/checkout`, `actions/configure-pages`, `actions/upload-pages-artifact`, `actions/deploy-pages`), triggered on push to `main` plus `workflow_dispatch`, with `permissions: contents: read, pages: write, id-token: write`.
3. Commit and push the workflow file.
4. Enable Pages with `build_type: "workflow"` on the repo. Prefer `gh api` if the `gh` CLI is installed and authenticated. If not, obtain a usable token via `git credential fill` for the `github.com` host (never print the token itself) and call the REST API directly with `curl`:
   - `POST /repos/{owner}/{repo}/pages` with `{"build_type":"workflow"}` to create it, or `PUT` the same path if it already exists (a 409 on POST means it exists — fall back to PUT).
5. Poll the workflow run (`GET /repos/{owner}/{repo}/actions/runs`) until it completes, then `curl` the resulting `https://{owner}.github.io/{repo}/` URL and confirm an HTTP 200 (not 404) **and** that the returned HTML actually looks like this project's page (e.g. check the `<title>`), not a generic GitHub 404 or an unrelated placeholder.
6. If the run fails or the page still 404s after a reasonable wait, report the actual failure — don't report success speculatively.

## Phase 4 — README

Write or update `README.md` at the repo root with real content about *this* project (read the actual code/files to describe it accurately — don't invent features). At minimum include: what the project is, how to run/open it, any hard constraints worth calling out (e.g. no backend, no persistence, single-file), and the live GitHub Pages link once it's known from Phase 3.

**Screenshot**: Capture a screenshot of the running app and embed it near the top of the README.

1. Prefer a Playwright MCP tool if one is available in this environment.
2. Otherwise fall back to a local headless browser (Edge or Chrome, both ship with Windows/most Linux desktops): launch it with `--headless --disable-gpu --window-size=1440,900 --screenshot=<path> file:///<absolute path to index.html or local dev URL>`. If the app needs a dev server/localhost URL instead of a raw file, start it first and wait for it to respond before screenshotting.
3. Save the image under `assets/` (e.g. `assets/screenshot.png`) in the repo, and reference it from the README with a relative path (`![...](assets/screenshot.png)`) so it renders on GitHub.
4. Open the captured image and visually confirm it actually shows the app's UI (not a blank page or an error) before embedding it — don't claim success speculatively.
5. If neither a Playwright tool nor a local browser executable is available, skip the screenshot, tell the user why, and continue with the rest of the README.

Commit and push the README (and screenshot) change.

## Phase 5 — About section (description + link)

Update the repo's About panel via the GitHub API (`PATCH /repos/{owner}/{repo}` with `description` and `homepage` fields), setting `homepage` to the GitHub Pages URL confirmed live in Phase 3, and a short one-line `description` summarizing the project. Use the same token-retrieval approach as Phase 3 (prefer `gh api` if available, else `git credential fill` + `curl`).

## Wrap-up

Report back concisely: what the security scan found (or that it was clean), the repo URL, the live Pages URL (confirmed non-404), and what was written/updated (workflow, README, About section). If any phase was blocked or skipped, say so explicitly rather than implying full success.
