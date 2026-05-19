# AI Changelog Updater — Setup & Usage Guide

Every merge to `main` automatically generates a formatted `CHANGELOG.md` entry using AI based on PR description. The workflow reads the PR title, body, labels, branch name, commit messages, and changed files, then produces a clean release-note style entry grouped under `Added / Changed / Fixed / Removed`.

---

## How it works

```
PR merged to main
       │
       ▼
Workflow triggers (pull_request: closed + merged == true)
       │
       ▼
Fetch PR metadata (files, commits, labels)
       │
       ▼
Build prompt → call Gemini API
       │
       ▼
Write entry to CHANGELOG.md
       │
       ▼
Push to short-lived branch  →  open PR  →  auto-merge  →  delete branch
```

The bot never pushes directly to `main`. It always opens a `changelog/pr-N` branch, creates a PR, and auto-merges it — fully compatible with branch protection rules and rulesets that require PRs.

---

## One-time repository setup

Complete all four steps before the workflow will succeed.

### 1. Add the Gemini API key as a organisation or repository secret

| Setting | Value |
|---|---|
| Secret name | `GEMINI_API_KEY` |
| Where | `Settings → Secrets and variables → Actions → New repository secret or organisation secret` |
| Value | Your Google AI Studio API key |

To obtain a key: [https://aistudio.google.com/apikey](https://aistudio.google.com/apikey)

---

### 2. Allow GitHub Actions to create and approve pull requests

> **Settings → Actions → General → Workflow permissions**

Check the box:
> ✅ **Allow GitHub Actions to create and approve pull requests**

Then click **Save**.

Without this, the workflow will fail with:
```
GraphQL: GitHub Actions is not permitted to create or approve pull requests
```

---

### 3. Enable auto-merge on the repository

> **Settings → General → Pull Requests section**

Check the box:
> ✅ **Allow auto-merge**

This allows the bot to call `gh pr merge --auto` so the changelog PR merges itself as soon as required checks pass, without any human intervention.

---

### 4. Add GitHub Actions to your ruleset bypass list (if you use rulesets)

If the repository has a ruleset with **"Require a pull request before merging"** enabled:

> **Settings → Rules → Rulesets → [your ruleset] → Bypass list → Add bypass**

Add:
| Actor type | Actor | Bypass mode |
|---|---|---|
| Role | GitHub Actions | Always allow |

Without this, direct pushes from workflows will be rejected. The PR-based flow in this workflow is already compliant, but the bypass is needed for the auto-merge to proceed without additional reviewer requirements.

> **Note:** If your ruleset also has **"Require status checks to pass"**, ensure the `changelog/pr-*` branches are either excluded from that rule, or that no required checks are configured — otherwise auto-merge will wait indefinitely.

---


## *** Extension to all repositories in an organization


## Reusable workflow (central `actions` repo)

Place workflow file in reusable actions workflow repository, example `<org-name>/actions/.github/workflows/ai-changelog-updater.yml`.

---

## Combining with an existing workflow

If your repo already has a build workflow (e.g. calling `<org-name>/actions/.github/workflows/my-build-workflow.yaml`), you can run changelog generation as an additional parallel job — no changes to the build job needed.

```yaml
name: Build and changelog

on:
  # Existing build trigger
  workflow_dispatch:
    inputs:
      environment:
        type: environment
        description: 'Environment'
        default: 'test'

  # Changelog trigger — fires only on merged PRs to main
  pull_request:
    types: [closed]
    branches:
      - main

jobs:
  # ── Existing build job (unchanged) ────────────────────────────
  service:
    if: github.event_name == 'workflow_dispatch'
    uses: <org-name>/actions/.github/workflows/my-build-workflow.yaml@main
    secrets: inherit
    with:
      service_name: my-serverless-function

  # ── Changelog job (new) ────────────────────────────────────────
  changelog:
    if: |
      github.event_name == 'pull_request' &&
      github.event.pull_request.merged == true &&
      github.event.pull_request.user.login != 'github-actions[bot]'
    uses: <org-name>/actions/.github/workflows/ai-changelog-updater.yml@main
    secrets: inherit
```

The `if:` conditions on each job ensure they are mutually exclusive — the build job only runs on `workflow_dispatch`, the changelog job only runs on merged PRs. Both live in one file with no interference.

---

## What the generated CHANGELOG.md looks like

```markdown
# Changelog

<!-- CHANGELOG_ENTRIES_START -->

## 2025-06-14 — [PR #42](https://github.com/<org-name>/my-repo/pull/42)

### Added
- New dark mode toggle in user preferences screen (PROJ-881)

### Fixed
- Crash on empty list when no results returned from search API

## 2025-06-10 — [PR #41](https://github.com/<org-name>/my-repo/pull/41)

### Changed
- Upgraded dependency versions across build configuration
```

The sentinel comment `<!-- CHANGELOG_ENTRIES_START -->` is the insertion point. New entries are always prepended at the top, keeping the most recent changes first. The sentinel is created automatically on first run.


---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `GraphQL: GitHub Actions is not permitted to create or approve pull requests` | PR creation permission not enabled | Step 2 above |
| `remote: error: GH013: Repository rule violations` | Direct push blocked by ruleset | Step 4 above — add bypass actor |
| `rejected (fetch first)` on `changelog/pr-N` branch | Stale branch from a previous failed run | Workflow handles this automatically with `--force` push; re-run the job |
| `Gemini API error: API key not valid` | Secret missing or wrong | Step 1 above — verify `GEMINI_API_KEY` secret |
| `auto-merge` PR never merges | Required status checks not passing on changelog branch | Exclude `changelog/pr-*` branches from required checks in your ruleset |
| Changelog updated on bot PRs (loop) | `user.login` check missing | Ensure the `if:` condition includes `github.event.pull_request.user.login != 'github-actions[bot]'` |

---

## Permissions summary

| Permission | Where to set | Required for |
|---|---|---|
| `GEMINI_API_KEY` secret | Repo → Settings → Secrets → Actions | Calling the Gemini API |
| Allow Actions to create PRs | Repo → Settings → Actions → General | `gh pr create` |
| Allow auto-merge | Repo → Settings → General | `gh pr merge --auto` |
| Ruleset bypass (if applicable) | Repo → Settings → Rules → Rulesets | Auto-merge completing |
| `pull-requests: write` in workflow | Workflow `permissions:` block | `gh pr create` and merge |
| `contents: write` in workflow | Workflow `permissions:` block | Committing and pushing `CHANGELOG.md` |