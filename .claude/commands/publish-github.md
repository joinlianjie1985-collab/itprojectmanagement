---
description: Scan for secrets, then push the project to GitHub, deploy it via GitHub Pages, and update the README and repo About
argument-hint: [github repo URL or owner/name] (optional if a remote already exists)
allowed-tools: Bash(git *), Bash(gh *), Bash(grep *), Bash(rg *), Bash(ls *), Bash(cat *), Bash(find *), Read, Write, Edit, Glob, Grep
---

# Publish this project to GitHub

Take this project from the working tree to a live, auto-redeploying GitHub Pages site,
with a README and a repo description that links to it — and refuse to push anything
containing secrets.

Target repo (may be empty): **$ARGUMENTS**

## Ground rules

- **The secret scan runs first, before anything leaves this machine.** A push is
  effectively irreversible — history, forks and GitHub's caches keep it — so scanning
  after the push would be pointless. If the scan finds something, stop and report;
  do not push and then clean up.
- **Never invent a repo URL.** If `$ARGUMENTS` is empty and there is no `origin`
  remote, ask the user for the repo URL or `owner/name` and stop until they answer.
- Respect any hard constraints in `CLAUDE.md` (build steps, file layout, dependencies).
  The publish workflow must not violate them.
- Do not force-push, rewrite history, or delete remote branches.
- Show the user what you are about to commit and push before you do it.

## Step 0 — Situational awareness

Run these and read the results before doing anything else:

```
git status --short
git branch --show-current
git remote -v
git log --oneline -5
ls -la
ls .github/workflows 2>/dev/null
gh --version 2>/dev/null || echo "gh CLI not installed"
```

Note which of these already exist, because every later step is create-**or**-edit:
a remote, a Pages workflow, a `README.md`, a repo description.

If `gh` is not installed, you can still do everything with plain `git` — but the repo
**About** section (step 5) has no CLI path. In that case prepare the exact text and
give the user click-by-click instructions instead of guessing.

## Step 1 — Scan for sensitive data (blocking)

Scan **tracked and untracked files that would be pushed** — not `.git/`, not
`node_modules/`. Look for:

- **Credentials in file contents**: API keys and tokens (`sk-`, `sk-ant-`, `ghp_`,
  `gho_`, `github_pat_`, `AKIA`, `ASIA`, `AIza`, `xox[baprs]-`, `glpat-`),
  `-----BEGIN ... PRIVATE KEY-----`, JWTs, connection strings with inline passwords
  (`postgres://user:pass@`, `mongodb+srv://`), and assignments matching
  `(password|passwd|secret|token|api[_-]?key|client[_-]?secret)\s*[:=]\s*` with a
  non-placeholder value.
- **Sensitive files**: `.env*` (except `.env.example`), `*.pem`, `*.key`, `*.p12`,
  `*.pfx`, `id_rsa*`, `*.keystore`, `credentials.json`, `service-account*.json`,
  `*.sqlite`/`*.db` with real data, cloud config like `.aws/`, `.npmrc`, `.netrc`.
- **Personal data**: real names, NRIC/FIN numbers, phone numbers, home addresses,
  personal email addresses, and any customer or trainee records. This project is a
  *fictional* demo — real personal data does not belong in it.
- **Internal detail that should not be public**: internal hostnames, private IP
  ranges, VPN or jump-host names, ticket links behind a corporate SSO, real employer
  or client names presented as real.

A useful starting sweep (tune as needed):

```
git ls-files -co --exclude-standard \
  | grep -vE '^(\.git/|node_modules/|_site/)' \
  | xargs grep -nEI '(sk-ant-|sk-[A-Za-z0-9]{20}|ghp_|gho_|github_pat_|AKIA|ASIA|AIza|xox[baprs]-|glpat-|BEGIN [A-Z ]*PRIVATE KEY|(password|passwd|secret|token|api[_-]?key|client[_-]?secret)[[:space:]"]*[:=])' 2>/dev/null
```

Then **read every hit** and judge it. Most matches are false positives — a variable
named `token` in a parser, the word "password" in a comment, a placeholder like
`YOUR_API_KEY_HERE`. Report only what is genuinely a live secret or real personal data.

Also check what is *already* committed, since the working tree is not the whole story:

```
git log --all --oneline -- '*.env' '*.pem' '*.key' 'id_rsa*' 2>/dev/null
```

**Outcome:**

- **Clean** → say so in one line and continue.
- **Findings** → STOP. List each finding as `file:line`, what it is, and the fix
  (delete it, move it to an untracked `.env`, replace with a placeholder, add to
  `.gitignore`). If a secret is already in a *pushed* commit, say plainly that
  rotating the credential is the only real remedy — removing it from history does
  not un-leak it. Do not push until the user has resolved the findings.

Ensure a `.gitignore` exists covering `.env`, `*.pem`, `*.key`, `.DS_Store`, and any
local build output. Create or extend it if needed.

## Step 2 — Upload the code to GitHub

Work out the target from `$ARGUMENTS` and the existing `origin`:

- **`origin` exists and matches / no argument given** → use it.
- **No `origin`** → accept a full URL (`https://github.com/owner/name.git`) or
  `owner/name`. Add it: `git remote add origin <url>`.
- **`origin` exists but the argument points somewhere else** → do not silently
  re-point it. Show the user both URLs and ask which they want.
- **Repo does not exist on GitHub yet** → with `gh`:
  `gh repo create <owner/name> --public --source=. --remote=origin`
  (ask public vs private first — never assume public). Without `gh`, tell the user to
  create the empty repo in the browser, then continue.

Then:

1. Stage the intended files — review `git status` first; do not blind-add junk.
2. Commit with a clear, factual message describing what actually changed.
3. Push: `git push -u origin main` (use the actual current branch name).
4. If the push is rejected as non-fast-forward, `git pull --rebase` and retry.
   Never resolve it with `--force`.

## Step 3 — GitHub Pages via GitHub Actions

If `.github/workflows/` already has a Pages workflow, read it and edit rather than
adding a second one. Otherwise create `.github/workflows/deploy-pages.yml`:

- Trigger on `push` to the default branch, plus `workflow_dispatch`.
- Least-privilege permissions: `contents: read`, `pages: write`, `id-token: write`.
- `concurrency: { group: pages, cancel-in-progress: false }`.
- Job steps: `actions/checkout@v4` → `actions/configure-pages@v5` → assemble the
  publishable files into `_site/` → `actions/upload-pages-artifact@v3` →
  `actions/deploy-pages@v4`, with `environment: github-pages` and
  `url: ${{ steps.deployment.outputs.page_url }}`.
- **Publish only what should be served.** Project docs (`CLAUDE.md`), the workflow
  directory and any assessment or internal files stay in the repo and out of `_site/`.
- If the project has a build step, run it here — but if `CLAUDE.md` says the project
  is a no-build single file, just copy the file. Do not introduce a toolchain.

Then enable Pages with the **GitHub Actions** source (not "deploy from a branch"):

```
gh api -X POST repos/<owner>/<name>/pages -f build_type=workflow 2>/dev/null \
  || gh api -X PUT repos/<owner>/<name>/pages -f build_type=workflow
```

Without `gh`, tell the user: **Settings → Pages → Source → GitHub Actions**.

Commit and push the workflow, then confirm the run:

```
gh run list --limit 3
gh run watch --exit-status   # optional, if a run is in flight
```

The site URL is `https://<owner>.github.io/<repo>/`. Report it, and note that the
first deployment can take a couple of minutes.

## Step 4 — README

Create or update `README.md` so it reflects what is actually in the repo — read the
code first, never describe features from assumption. Include:

- Project title and a one-or-two-sentence description of what it does.
- **A live demo link to the Pages URL, near the top.**
- How to run it locally (for a `file://` project, say plainly that you open the file).
- Key features, kept honest and short.
- Tech notes and any deliberate constraints (e.g. vanilla JS, single file, no build,
  no persistence — say *why* if `CLAUDE.md` explains it).
- A note that it is a demo with fictional data, if that is true.

Do not add badges for CI that does not exist, or a licence the user has not chosen.
If `README.md` already has content the user wrote, preserve their wording and edit
around it.

## Step 5 — Repo About + Pages link

Set the repo description, homepage and topics.

With `gh`:

```
gh repo edit <owner>/<name> \
  --description "<one line, under ~120 chars, what the project is>" \
  --homepage "https://<owner>.github.io/<repo>/" \
  --add-topic <topic> --add-topic <topic>
```

Verify:

```
gh repo view <owner>/<name> --json description,homepageUrl,repositoryTopics
```

Without `gh`, print the exact description, homepage URL and topics you propose, and
tell the user: repo main page → **About** (gear icon) → paste description, paste the
Pages URL into *Website*, tick *Use your GitHub Pages website*, add topics, Save.

## Step 6 — Report

Finish with a short summary:

- Secret scan: clean, or what was found and fixed.
- Repo URL and the commit(s) pushed.
- Pages URL, and whether the deployment run actually succeeded.
- README and About: created or updated.
- Anything the user must still do by hand (enabling Pages, setting About without `gh`,
  rotating a leaked credential).

State failures plainly. If the Actions run failed, say so and paste the relevant log
lines rather than reporting success.
