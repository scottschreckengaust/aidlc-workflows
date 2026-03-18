# Administrative Guide

This guide documents the CI/CD infrastructure, GitHub Workflows, protected environments, secrets, variables, permissions, and release process for the `awslabs/aidlc-workflows` repository.

**Audience:** Repository administrators, maintainers, and AI coding agents working on this repository.

**Related documentation:**
- [Developer's Guide](DEVELOPERS_GUIDE.md) — Running builds locally (CodeBuild + `act`)
- [Contributing Guidelines](../CONTRIBUTING.md) — Contribution process and conventions
- [README](../README.md) — User-facing setup and usage

---

## Table of Contents

- [Repository Overview](#repository-overview)
- [CI/CD Architecture](#cicd-architecture)
- [Workflow Reference](#workflow-reference)
  - [CodeBuild Workflow](#codebuild-workflow-codebuildyml)
  - [Release Workflow](#release-workflow-releaseyml)
  - [Changelog Workflow](#changelog-workflow-changelogyml)
- [Protected Environments](#protected-environments)
- [Secrets and Variables](#secrets-and-variables)
- [Permissions Model](#permissions-model)
- [Security Posture](#security-posture)
- [Code Ownership](#code-ownership)
- [Release Process](#release-process)
- [Changelog Configuration](#changelog-configuration)

---

## Repository Overview

This repository publishes the **AI-DLC (AI-Driven Development Life Cycle)** methodology as a set of markdown rule files under `aidlc-rules/`. The CI/CD infrastructure handles:

- **Continuous integration** via AWS CodeBuild (evaluation and reporting)
- **Release distribution** via GitHub Releases (zipped rule files)
- **Changelog generation** via git-cliff (automated PRs from conventional commits)

```
awslabs/aidlc-workflows/
├── .github/
│   ├── CODEOWNERS
│   ├── ISSUE_TEMPLATE/           # Bug, feature, RFC, docs templates
│   └── workflows/
│       ├── codebuild.yml         # CI via AWS CodeBuild
│       ├── release.yml           # GitHub Release on tag push
│       └── changelog.yml         # Changelog PR after release
├── aidlc-rules/                  # The distributable product
│   ├── aws-aidlc-rules/          # Core workflow rules
│   └── aws-aidlc-rule-details/   # Detailed rules by phase
├── cliff.toml                    # git-cliff changelog configuration
├── docs/
│   ├── ADMINISTRATIVE_GUIDE.md   # This file
│   └── DEVELOPERS_GUIDE.md       # Local build instructions
└── scripts/
    └── aidlc-evaluator/          # Evaluation framework (in development)
```

---

## CI/CD Architecture

Three workflows form two distinct pipelines:

### Pipeline 1: Release (draft → review → publish)

```
git push tag v* ──┬──→ release.yml (creates DRAFT release with ai-dlc-rules zip)
                  │
                  └──→ codebuild.yml (runs CodeBuild on tag, uploads artifacts to draft)
                            │
                  ┌─────────┘
                  ▼
        Human reviews draft release in GitHub UI
                  │
                  ▼ (publishes the release)
                  │
                  └──→ changelog.yml (opens PR with updated CHANGELOG.md)
```

Both `release.yml` and `codebuild.yml` trigger independently on the same `v*` tag push. The `codebuild.yml` workflow handles all release states resiliently:
- **Draft exists** (normal case) — attaches build artifacts to the draft
- **No release yet** (codebuild finished first) — creates a draft with build artifacts
- **Already published** (re-run) — attempts to replace artifacts, warns gracefully if immutable

### Pipeline 2: Continuous Integration

```
git push main ────→ codebuild.yml (runs CodeBuild, uploads workflow artifacts)
workflow_dispatch ─→ codebuild.yml
```

The `changelog.yml` workflow triggers on `release: published` (not `created`), so draft releases do not trigger changelog generation prematurely.

---

## Workflow Reference

### CodeBuild Workflow (`codebuild.yml`)

| Property | Value |
|----------|-------|
| **File** | `.github/workflows/codebuild.yml` |
| **Triggers** | `push` to `main`, `push` tags `v*`, `workflow_dispatch` (manual) |
| **Environment** | `codebuild` (protected, manual approval) |
| **Runner** | `ubuntu-latest` |
| **Concurrency** | Groups by `{workflow}-{ref}`, cancels in-progress |

**Purpose:** Runs an AWS CodeBuild project, downloads primary and secondary artifacts from S3, caches them in GitHub Actions cache, uploads them as workflow artifacts, and (on tag pushes) attaches them to the GitHub Release.

**Job: `build`**

| Step | Name | Condition | Action |
|------|------|-----------|--------|
| 1 | List caches | _(always)_ | `gh cache list` for existing project caches |
| 2 | Check cache | _(always)_ | `actions/cache/restore` with `lookup-only: true` |
| 3 | Configure AWS credentials | cache miss | `aws-actions/configure-aws-credentials` (OIDC) |
| 4 | Run CodeBuild | cache miss | `aws-actions/aws-codebuild-run-build` with inline buildspec |
| 5 | Build ID | cache miss (always) | Echo CodeBuild build ID |
| 6 | Download CodeBuild artifacts | cache miss | Download primary + secondary artifacts from S3 |
| 7 | List CodeBuild artifacts | cache miss | List and inspect downloaded zip files |
| 8 | Clean old report caches | cache miss | Delete 3 oldest matching caches for branch |
| 9 | Save report to cache | cache miss | `actions/cache/save` with key `{project}-{branch}-{sha}` |
| 10 | Upload primary artifact | `!env.ACT` | `actions/upload-artifact` for `{project}.zip` |
| 11 | Upload evaluation artifact | `!env.ACT` | `actions/upload-artifact` for `evaluation.zip` |
| 12 | Upload trend artifact | `!env.ACT` | `actions/upload-artifact` for `trend.zip` |
| 13 | Upload artifacts to release | tag push (`v*`) | Attach build artifacts to GitHub Release (draft or published) |

**Caching strategy:** The cache key `{project}-{branch}-{sha}` ensures that the same commit on the same branch is never built twice. On cache hit, steps 3–9 are skipped entirely.

**Inline buildspec:** The workflow embeds a full `buildspec-override` rather than referencing an external file. The buildspec:
- Installs `gh` CLI (via dnf) and `uv` (Python package manager)
- Determines build context: release (tagged), pre-release (default branch), or pre-merge (feature branch)
- Creates placeholder evaluation and trend report files under `.codebuild/`
- Outputs a primary artifact (all files under `.codebuild/`) and two secondary artifacts (`evaluation`, `trend`)

**Artifact upload compatibility:** Upload steps are gated by `!env.ACT` because `actions/upload-artifact` v6 is incompatible with the [`act`](https://github.com/nektos/act) local runner.

**External actions (all SHA-pinned):**

| Action | Version | SHA |
|--------|---------|-----|
| `actions/cache/restore` | v5.0.3 | `cdf6c1fa76f9f475f3d7449005a359c84ca0f306` |
| `aws-actions/configure-aws-credentials` | v6.0.0 | `8df5847569e6427dd6c4fb1cf565c83acfa8afa7` |
| `aws-actions/aws-codebuild-run-build` | v1.0.18 | `d8279f349f3b1b84e834c30e47c20dcb8888b7e5` |
| `actions/cache/save` | v5.0.3 | `cdf6c1fa76f9f475f3d7449005a359c84ca0f306` |
| `actions/upload-artifact` | v6.0.0 | `b7c566a772e6b6bfb58ed0dc250532a479d7789f` |

---

### Release Workflow (`release.yml`)

| Property | Value |
|----------|-------|
| **File** | `.github/workflows/release.yml` |
| **Trigger** | `push` on tags matching `v*` |
| **Environment** | _(none)_ |
| **Runner** | `ubuntu-latest` |

**Purpose:** Creates a **draft** GitHub Release with a zip of `aidlc-rules/` when a version tag is pushed. The release is kept as a draft so that CodeBuild artifacts can be attached and reviewed before publishing.

**Job: `release` ("Create Release")**

| Step | Name | Action |
|------|------|--------|
| 1 | Checkout code | `actions/checkout` with `fetch-depth: 0` |
| 2 | Extract version | Parse `GITHUB_REF` into `version` (no `v`) and `tag` (with `v`) |
| 3 | Create release artifact | `zip -r ai-dlc-rules-v{VERSION}.zip aidlc-rules/` |
| 4 | Create GitHub Release | `softprops/action-gh-release` with `draft: true` and zip attached |

**Release naming:** `AI-DLC Workflow v{VERSION}` (e.g., `AI-DLC Workflow v0.1.6`)

**External actions (SHA-pinned):**

| Action | Version | SHA |
|--------|---------|-----|
| `actions/checkout` | v6.0.1 | `8e8c483db84b4bee98b60c0593521ed34d9990e8` |
| `softprops/action-gh-release` | v2.5.0 | `a06a81a03ee405af7f2048a818ed3f03bbf83c7b` |

---

### Changelog Workflow (`changelog.yml`)

| Property | Value |
|----------|-------|
| **File** | `.github/workflows/changelog.yml` |
| **Trigger** | `release` event, type `published` |
| **Environment** | _(none)_ |
| **Runner** | `ubuntu-latest` |

**Purpose:** After a release is published, generates `CHANGELOG.md` from conventional commits using git-cliff and opens a PR.

**Job: `changelog` ("Update Changelog")**

| Step | Name | Action |
|------|------|--------|
| 1 | Checkout code | `actions/checkout` with `fetch-depth: 0` |
| 2 | Generate changelog | `orhun/git-cliff-action` with `cliff.toml` config |
| 3 | Create Pull Request | Create branch `changelog/{tag}`, commit, push, `gh pr create` with label `documentation` |

The PR is authored by `github-actions[bot]` and titled `docs: update changelog for {tag}`.

**External actions (SHA-pinned):**

| Action | Version | SHA |
|--------|---------|-----|
| `actions/checkout` | v6.0.1 | `8e8c483db84b4bee98b60c0593521ed34d9990e8` |
| `orhun/git-cliff-action` | v4.7.0 | `e16f179f0be49ecdfe63753837f20b9531642772` |

---

## Protected Environments

| Environment | Used By | Purpose |
|-------------|---------|---------|
| `codebuild` | `codebuild.yml` job `build` | Gates access to AWS credentials for CodeBuild |

The `codebuild` environment is the only protected environment. It contains:
- The `AWS_CODEBUILD_ROLE_ARN` secret (required for OIDC-based AWS role assumption)
- Possibly the repository variables `CODEBUILD_PROJECT_NAME`, `AWS_REGION`, and `ROLE_DURATION_SECONDS` (these may alternatively be set at the repository level)

Environment protection rules (configured in GitHub repository settings) may include required reviewers or deployment branch restrictions.

---

## Secrets and Variables

### Secrets

| Secret | Scope | Used By | Purpose |
|--------|-------|---------|---------|
| `AWS_CODEBUILD_ROLE_ARN` | Environment (`codebuild`) | `codebuild.yml` | IAM Role ARN for OIDC-based AWS STS role assumption |
| `GITHUB_TOKEN` | Automatic (GitHub-provided) | `release.yml`, `changelog.yml` | Authenticate GitHub API calls (release creation, PR creation) |

The `codebuild.yml` workflow also uses `github.token` (the automatic token, accessed without the `secrets.` prefix) for cache management and release asset uploads.

### Repository Variables

| Variable | Used By | Default Fallback | Purpose |
|----------|---------|------------------|---------|
| `CODEBUILD_PROJECT_NAME` | `codebuild.yml` | `codebuild-project` | AWS CodeBuild project name |
| `AWS_REGION` | `codebuild.yml` | `us-east-1` | AWS region for CodeBuild and STS |
| `ROLE_DURATION_SECONDS` | `codebuild.yml` | `7200` | STS session duration (seconds) |

All three variables have sensible defaults via `${{ vars.VAR || 'default' }}` syntax, so the workflow runs even without explicit variable configuration.

---

## Permissions Model

### Workflow-level permissions

| Workflow | Permissions |
|----------|-------------|
| `codebuild.yml` | All 16 scopes explicitly set to `none` |
| `release.yml` | `contents: write` |
| `changelog.yml` | `contents: write`, `pull-requests: write` |

### Job-level permissions (overrides)

| Workflow | Job | Permissions | Rationale |
|----------|-----|-------------|-----------|
| `codebuild.yml` | `build` | `actions: write`, `contents: write`, `id-token: write` | Cache management, release asset upload, OIDC token for AWS STS |

The `codebuild.yml` workflow follows a **deny-all-then-grant** pattern: every permission scope is set to `none` at the workflow level, then only the 3 required scopes are granted at the job level. This is the strictest possible configuration and prevents privilege escalation from compromised steps.

---

## Security Posture

| Control | Implementation |
|---------|----------------|
| **Supply-chain protection** | All external actions pinned to full commit SHAs (not mutable version tags) |
| **AWS authentication** | OIDC-based role assumption via `id-token: write` — no static credentials stored |
| **Least-privilege tokens** | `codebuild.yml` explicitly denies all 16 permission scopes at workflow level, grants only 3 at job level |
| **Environment protection** | `codebuild` environment gates AWS credential access with potential reviewer/branch rules |
| **Concurrency control** | `codebuild.yml` cancels in-progress runs for the same branch |
| **Code ownership** | `.github/` (including workflows) owned exclusively by `@awslabs/aidlc-admins` via CODEOWNERS |
| **Account masking** | `mask-aws-account-id: true` in AWS credential configuration |

---

## Code Ownership

Defined in `.github/CODEOWNERS`:

| Path | Owners |
|------|--------|
| `*` (default) | `@awslabs/aidlc-admins` `@awslabs/aidlc-maintainers` |
| `.github/` | `@awslabs/aidlc-admins` |
| `.github/CODEOWNERS` | `@awslabs/aidlc-admins` |
| `aidlc-rules/` | `@awslabs/aidlc-admins` `@awslabs/aidlc-maintainers` `@awslabs/aidlc-writers` |
| `assets/` | `@awslabs/aidlc-admins` `@awslabs/aidlc-maintainers` `@awslabs/aidlc-writers` |
| `scripts/` | `@awslabs/aidlc-admins` `@awslabs/aidlc-maintainers` |
| `CHANGELOG.md`, `cliff.toml`, `LICENSE`, etc. | `@awslabs/aidlc-admins` |

**Key implication:** Only `@awslabs/aidlc-admins` can approve changes to `.github/` (workflows, CODEOWNERS, issue templates).

---

## Release Process

Releases are tag-driven with no separate version file. The process uses draft releases for human review before publishing.

1. **Create and push a version tag:**
   ```bash
   git tag v1.2.0
   git push origin v1.2.0
   ```

2. **`release.yml` runs automatically:**
   - Zips `aidlc-rules/` into `ai-dlc-rules-v1.2.0.zip`
   - Creates a **draft** GitHub Release named "AI-DLC Workflow v1.2.0" with the zip attached

3. **`codebuild.yml` runs automatically** (requires `codebuild` environment approval):
   - Runs CodeBuild on the tagged commit
   - Downloads build artifacts (primary, evaluation, trend)
   - Attaches artifacts to the draft release (or creates a draft if one doesn't exist yet)

4. **Review the draft release** in the GitHub UI:
   - Verify all expected artifacts are attached (rules zip + build artifacts)
   - Review release notes and edit if needed

5. **Publish the release** by clicking "Publish release" in the GitHub UI.

6. **`changelog.yml` runs automatically** (triggered by the release `published` event):
   - Generates `CHANGELOG.md` from all conventional commits using git-cliff
   - Opens a PR on branch `changelog/v1.2.0` with label `documentation`

7. **Merge the changelog PR** to complete the release cycle.

**Note:** The `codebuild` protected environment may need its deployment branch rules updated to allow `v*` tags (in addition to `main`) for tag-triggered builds to proceed.

---

## Changelog Configuration

Defined in `cliff.toml` (used by `changelog.yml`):

| Setting | Value |
|---------|-------|
| **Commit format** | Conventional commits (`feat:`, `fix:`, `docs:`, etc.) |
| **Tag pattern** | `v[0-9].*` |
| **Sort order** | Oldest first |

**Commit groups:**

| Prefix | Group Name |
|--------|------------|
| `feat` | Features |
| `fix` | Bug Fixes |
| `doc` | Documentation |
| `perf` | Performance |
| `refactor` | Refactoring |
| `style` | Style |
| `test` | Tests |
| `ci` | CI/CD |
| `chore` | Miscellaneous |

Unconventional commits are filtered out (`filter_unconventional = true`).
