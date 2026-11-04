---
description: Security-scan, push to GitHub, write README + About, deploy GitHub Pages, and set up CI/CD
argument-hint: "[owner/repo or GitHub URL] [public|private]"
allowed-tools: Bash, PowerShell, Read, Write, Edit, Glob, Grep
---

# Publish this project to GitHub

Repo info from the user: `$ARGUMENTS`

Work through the steps below in order. Stop and report if any step fails; do not skip ahead.
Read `CLAUDE.md` first: its hard constraints (single `index.html`, no persistence, the only
network call is FormSubmit, neutral branding) apply to everything you write here.

## 0. Resolve the target repo

- Parse `$ARGUMENTS` for `owner/repo` (or a `https://github.com/owner/repo(.git)` URL) and an
  optional `public` / `private` visibility. Default visibility is `public` (GitHub Pages on a
  free plan needs a public repo).
- If no repo was given, check `git remote get-url origin`. If there is none either, ask the
  user for the repo with AskUserQuestion. Never guess an owner.
- Check prerequisites: `git --version`, `gh --version`, `gh auth status`. If `gh` is not
  logged in, tell the user to run `gh auth login` and stop.
- Check whether the repo exists: `gh repo view OWNER/REPO --json name,visibility,url`.

## 1. Security scan (before anything leaves the machine)

The goal is that nothing sensitive is pushed to GitHub or published on the Pages site.
Scan every file that would be committed (`git ls-files` plus untracked, non-ignored files from
`git ls-files --others --exclude-standard`) and the full git history (`git log -p --all`).

Check for:

1. **Secrets / credentials:** API keys and tokens (`ghp_`, `github_pat_`, `gho_`, `sk-`,
   `sk-ant-`, `AKIA`, `AIza`, `xox[baprs]-`, `-----BEGIN .* PRIVATE KEY-----`), `password=`,
   `secret=`, `token=`, bearer headers, connection strings, `.env` files, `*.pem`, `*.key`,
   `*.p12`, `*.pfx`, `id_rsa*`. If `gitleaks` is installed, also run
   `gitleaks detect --source . --redact -v`.
2. **Personal data:** real email addresses and phone numbers. In particular check
   `FORMSUBMIT_ENDPOINT` in `index.html`:
   - The placeholder `YOUR_EMAIL@example.com` is fine.
   - If it holds a real address, warn the user that it will be public in the repo and on the
     live page, and recommend FormSubmit's random-string alias endpoint
     (`https://formsubmit.co/ajax/<random-id>`, shown in the activation email) instead.
     Ask before continuing.
3. **Outbound network calls in `index.html`:** grep for `fetch(`, `XMLHttpRequest`,
   `sendBeacon`, `WebSocket`, `EventSource`, `new Image(`, `import(`, `<script src`,
   `<link href="http`, `@import`, `url(http`, and any `http://` / `https://` URL. The only
   allowed call is `notifyNewTask()` → `FORMSUBMIT_ENDPOINT`. Also confirm there is no
   `localStorage`, `sessionStorage`, `indexedDB` or `document.cookie`, and no analytics or
   tracking script.
4. **What the payload sends:** read `notifyNewTask()` and list exactly which fields it POSTs.
   It must send only task fields the user typed into the form, nothing from the browser
   environment (user agent, location, other tasks, etc.).
5. **Files that should not be public:** local tool folders (`.playwright-mcp/`,
   `.claude/settings.local.json`), OS junk, large binaries. Make sure `.gitignore` covers
   them; add entries if missing. Keep `.claude/commands/` committed.
6. **What Pages will serve:** the Pages site must publish only `index.html` (see step 5), not
   `CLAUDE.md`, `.github/`, `.claude/` or `docs/`.

If you find a real secret: stop, show the file and line (redact the value), and tell the user
to rotate it. If it is in git history, explain that removing it from the latest commit is not
enough and ask how they want to proceed (for example `git filter-repo`). Do not push.

Print a short scan report: each check with PASS / WARN / FAIL.

## 2. README.md

Create `README.md`, or update it if it exists (keep accurate content and fix what is stale).
Write it for someone landing on the GitHub repo. Include:

- Title and one-line description (IT PMO Kanban board, a demo/training tool for a fictitious
  bank; not an official system).
- A **Live demo** link to the Pages URL `https://OWNER.github.io/REPO/` near the top.
- Screenshot `docs/screenshot.png` if it exists.
- Features (four columns, drag and drop plus the keyboard "Move ▸" menu, filters, overdue
  highlighting, add/delete, email notification on new task).
- How to run (open `index.html` in a browser; no build, no server).
- Configuration: `FORMSUBMIT_ENDPOINT` and FormSubmit's one-time activation email, plus the
  `file://` caveat from `CLAUDE.md`.
- Privacy: no storage; a refresh resets to seed data; the only network request is the
  FormSubmit notification and what it contains.
- Tech stack (vanilla HTML/CSS/JS, single file, no dependencies) and project structure.
- CI/CD section describing the workflows from step 6.
- License line only if a LICENSE file exists (ask before adding one).

## 3. Commit and push

- If there is no repo yet, create it:
  `gh repo create OWNER/REPO --public|--private --source . --remote origin`
  (without `--push`; push below). If `origin` exists but points somewhere else, show both
  URLs and ask before changing it.
- Make sure the branch is `main`.
- Stage specific files (never `git add -A` blindly): run `git status` and review what would
  be committed against the scan from step 1.
- Commit with a clear message ending with the attribution line from the system reminder.
- `git push -u origin main`. Never force-push unless the user explicitly says so.

## 4. GitHub About section

Set the repo description, website and topics:

```
gh repo edit OWNER/REPO \
  --description "IT PMO Kanban board: single-file vanilla HTML/CSS/JS demo for a fictitious bank" \
  --homepage "https://OWNER.github.io/REPO/" \
  --add-topic kanban --add-topic project-management --add-topic pmo \
  --add-topic vanilla-js --add-topic html --add-topic github-pages
```

Keep the description under 350 characters. Adjust the wording if the project has changed.

## 5. GitHub Pages

- Enable Pages with GitHub Actions as the source:
  `gh api -X POST repos/OWNER/REPO/pages -f build_type=workflow`
  (if it returns 409 it already exists; then
  `gh api -X PUT repos/OWNER/REPO/pages -f build_type=workflow`).
- The deploy workflow (`.github/workflows/pages.yml`) must upload a staged folder containing
  **only** `index.html` (and `docs/screenshot.png` only if the page itself uses it), never
  `path: .`. Stage it like this:

  ```yaml
  - name: Stage site
    run: |
      mkdir -p _site
      cp index.html _site/
      touch _site/.nojekyll
  - uses: actions/upload-pages-artifact@v3
    with:
      path: _site
  ```

- Get the URL: `gh api repos/OWNER/REPO/pages --jq .html_url`.

## 6. CI/CD with GitHub Actions

Create or update two workflows. Pin the major versions of actions (`@v4`, etc.) and give
each job the least `permissions` it needs.

**`.github/workflows/ci.yml`**, on `push` and `pull_request` to `main`:

- **Secret scan:** `gitleaks/gitleaks-action@v2` with `fetch-depth: 0` on checkout
  (`GITHUB_TOKEN` in env).
- **Constraint checks** (a bash step using `grep`, failing the job on a match):
  - `index.html` exists, and no other `.html`, `.js` or `.css` files are in the repo.
  - No `localStorage`, `sessionStorage`, `indexedDB`, `document.cookie`.
  - No `alert(` or `confirm(` calls.
  - No `!important`.
  - No external `<script src=`, `<link ... href="http`, `@import` or font/CDN URLs.
  - The only `fetch(` is inside `notifyNewTask`, and the only `https://` URL in script code is
    `FORMSUBMIT_ENDPOINT`.
- **HTML validation:** `npx --yes html-validate@8 index.html` (CI only; this does not add
  npm to the project). If the existing markup fails rules that are deliberate, add a minimal
  `.htmlvalidate.json` rather than changing the page.

**`.github/workflows/pages.yml`**: deploys on push to `main` and `workflow_dispatch`. Make it
run only after CI passes: either add the deploy as a job in `ci.yml` with `needs:` on the
check jobs, or trigger `pages.yml` with `workflow_run` on CI `completed` and
`if: github.event.workflow_run.conclusion == 'success'`. Keep `concurrency: pages`.

Commit the workflow changes and push.

## 7. Verify and add the Pages link to About

- Watch the runs: `gh run list --limit 5`, then `gh run watch <id> --exit-status` for CI
  and the Pages deploy. If a run fails, read `gh run view <id> --log-failed`, fix the cause,
  and push again.
- Confirm the site is live: `curl -sS -o /dev/null -w "%{http_code}" https://OWNER.github.io/REPO/`
  should return `200` (allow a minute or two after the first deploy).
- Set the About website to the live Pages URL (re-run
  `gh repo edit OWNER/REPO --homepage <pages-url>` with the URL from step 5) and confirm with
  `gh repo view OWNER/REPO --json homepageUrl,description,repositoryTopics`.
- Make sure the README's Live demo link matches that URL.

## 8. Screenshot the live site

Use the Playwright MCP tools (server configured in `.mcp.json`) to capture the deployed page,
not the local file, so the screenshot shows exactly what visitors see.

- `browser_resize` to 1440×900, then `browser_navigate` to the live Pages URL.
- `browser_take_screenshot` with `filename: docs/screenshot.png`, `type: png`,
  `scale: css` (viewport only, not full page). Then `browser_close`.
- Read the image back and check it: header, summary bar, filters and all four columns with
  seed cards should be visible, with no error toast. A favicon 404 in the console is harmless;
  any other console error is worth reporting.
- Make sure the README shows it near the top, under the Live demo link, with descriptive alt
  text: `![IT PMO Kanban board with ...](docs/screenshot.png)`.
- If the image changed, stage only `docs/screenshot.png` (and `README.md` if edited), commit
  ("Update README screenshot from live site" plus the attribution line) and push. Pages does
  not serve `docs/`, so this push needs no redeploy check beyond CI passing.

## 9. Report

Finish with a short summary:

- Repo URL and live Pages URL.
- Security scan results (PASS / WARN / FAIL per check), and anything the user must act on,
  for example activating FormSubmit or rotating a key.
- Files created or changed.
- Workflow run results with links.
