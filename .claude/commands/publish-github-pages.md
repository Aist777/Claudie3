---
description: Push this project to GitHub, enable Pages + CI/CD, set the repo About section, deploy, and run a security scan first
argument-hint: <owner/repo or GitHub URL> [public|private]
---

Publish this project (the single-file `index.html` demo) to GitHub and deploy it to GitHub Pages.

Repo info from the user: `$ARGUMENTS`

If `$ARGUMENTS` does not name a repository, stop and ask for the `owner/repo` (or URL). Do not guess the destination.

Read `CLAUDE.md` first and follow its constraints (single file, no frameworks/CDNs/storage APIs, `index.html` stays at the repo root, the FormSubmit endpoint is the only network call).

Work through the steps in order. Stop and report if any step fails; do not work around a failure by skipping a check.

## Step 1: Confirm the target
- Parse `owner/repo` from `$ARGUMENTS`. Default visibility is `public` (GitHub Pages on a free plan needs a public repo) unless the user gave `private`.
- Run `gh auth status`. If not logged in, stop and ask the user to run `gh auth login`.
- Check `git remote -v`. The current `origin` may point elsewhere. If it differs from the target repo, show both and ask before changing it. Never overwrite a remote without confirmation.
- Check `git config user.email`. Commit emails become public in history. Show it to the user and confirm it is the address they want published.

## Step 2: Security scan (before anything leaves the machine)
Scan only what would be published: tracked files plus anything staged for commit. Do this before any `git push`.

1. Inventory: `git status --short` and `git ls-files`. List every file that would be committed.
2. Untracked personal files: `.DS_Store`, personal images (for example `Han Wen Hsian.png`, `Screenshot*.png`), and personal documents (for example the `.docx`) must NOT be committed. Do not stage them. Add `.DS_Store` to `.gitignore` if it is not already there. Ask before committing any other untracked file.
3. Secret patterns: run `gitleaks detect --source . --no-git` if `gitleaks` is installed. If it is not installed, grep the publishable files for:
   - keys and tokens: `AKIA[0-9A-Z]{16}`, `ghp_[A-Za-z0-9]{36}`, `github_pat_`, `sk-[A-Za-z0-9]{20,}`, `xox[baprs]-`, `-----BEGIN .*PRIVATE KEY-----`
   - passwords and secrets: `password`, `passwd`, `secret`, `api[_-]?key`, `token`
   - personal data: email addresses (`[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}`), local paths (`/Users/`), phone numbers, and personal names from the user's files
   Any hit must be reviewed by hand. Report what was found and where, and do not publish until the user decides.
4. Network review of `index.html`: list every `http(s)://`, `fetch(`, `XMLHttpRequest`, `<script src`, `<link`, `<img src`, `<iframe`, and `form action`. The only allowed external call is the FormSubmit endpoint in `FORMSUBMIT_ENDPOINT`. Flag anything else.
5. Storage check: grep for `localStorage`, `sessionStorage`, `indexedDB`, `document.cookie`. None should appear.
6. Report the scan result in a short table (check, result, action). Continue only when there are no unresolved findings, or the user has accepted them explicitly.

## Step 3: Push the code
- Make sure `.gitignore` excludes `.DS_Store`.
- Stage only the project files that belong in the repo: `index.html`, `CLAUDE.md`, `.github/`, `.claude/commands/`, `.gitignore`, `README.md`. Stage nothing else without asking.
- Commit with a clear message, ending with the attribution line:
  `Co-Authored-By: Claude Haiku 5.5 <noreply@anthropic.com>`
- If the target repo does not exist, create it: `gh repo create <owner/repo> --<public|private> --source . --remote origin` (confirm the name and visibility first if they were not given explicitly). If it exists, add or update `origin` after confirmation.
- Ensure the branch is `main`, then `git push -u origin main`.

## Step 4: Create or edit the GitHub Page
The Pages site is `index.html` itself, served from the repo root through the workflow. There is no separate page to create.
- Check whether Pages is already enabled: `gh api repos/<owner/repo>/pages`.
- If not enabled, enable it with the GitHub Actions source: `gh api -X POST repos/<owner/repo>/pages -f build_type=workflow`.
- If enabled with a different source, change it to workflow: `gh api -X PUT repos/<owner/repo>/pages -f build_type=workflow`.
- Read the resulting `html_url` for the link used later.

## Step 5: Set up CI/CD
- Confirm `.github/workflows/deploy-pages.yml` exists. It already builds `index.html` and deploys on every push to `main`. Do not duplicate it.
- If the file is missing, recreate it with the same content: Pages build and deploy jobs, `actions/checkout@v4`, `actions/configure-pages@v5`, `actions/upload-pages-artifact@v3`, `actions/deploy-pages@v4`, triggers on `push` to `main` and `workflow_dispatch`, and `permissions: contents: read, pages: write, id-token: write`.
- Validate the YAML: `ruby -ryaml -e 'YAML.load_file(".github/workflows/deploy-pages.yml")'` or an equivalent. Fix any errors before pushing.

## Step 6: Edit the repo About section
- Description (one line): `gh repo edit <owner/repo> --description "IT PMO Kanban demo: a single-file, dependency-free board for IT project portfolio management."`
- Website (homepage): set it to the Pages URL from Step 4: `gh repo edit <owner/repo> --homepage "<pages html_url>"`.
- Optional topics if the user wants them: `gh repo edit <owner/repo> --add-topic kanban,pmo,html`.

## Step 7: Deploy
- If the Step 3 push already triggered the workflow, watch it: `gh run list --workflow=deploy-pages.yml --limit 1` and then `gh run watch <run-id> --exit-status`.
- If it did not trigger, run `gh workflow run deploy-pages.yml` and watch it the same way.
- If the run fails, read the log (`gh run view <run-id> --log-failed`), fix the cause, push again, and repeat until it passes.

## Step 8: Verify
- `curl -sS -o /dev/null -w "%{http_code}\n" <pages html_url>` must return `200`. Wait and retry for up to a few minutes, since Pages can take a short while to go live.
- Confirm the served page has the expected title: `curl -sS <pages html_url> | grep -i "<title>"`.
- Confirm the About section shows the description and the homepage link: `gh repo view <owner/repo> --json description,homepageUrl`.

## Final report
Give the user:
- the repo URL and the Pages URL
- the scan table from Step 2, including anything excluded from the commit
- the commit hash that was pushed and the workflow run result
- anything left undone, with the reason
