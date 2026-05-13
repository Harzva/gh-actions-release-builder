---
name: gh-actions-release-builder
description: Design, repair, and harden GitHub Actions workflows for CI, tests, build, packaging, artifacts, GitHub Releases, Pages deployment, and multi-platform software delivery. Use when Codex needs to create or update `.github/workflows/*.yml`, move local build commands into Actions, produce APK/EXE/ZIP/package artifacts, configure release-on-tag workflows, debug failed Actions, or combine build automation with `$gh-account-router` publishing.
---

# GitHub Actions Release Builder

Use this skill to make GitHub Actions the source of truth for building, packaging, testing, and releasing software. Prefer workflows that are reproducible, least-privilege, observable, and friendly to repositories where the local machine cannot compile the product.

## Core Rule

- Do not build or compile release software locally when the repository or user says the local environment is unavailable. Put build, package, artifact, and release steps in GitHub Actions.
- Local work is limited to editing workflow files, validating YAML shape, inspecting project metadata, and lightweight static checks that do not require the missing build environment.
- If publishing or pushing is required, combine this skill with `$gh-account-router` to select the correct GitHub account, token, remote, and owner.
- If product delivery decisions are broader than CI/CD, combine this skill with `$software-dev-pipeline` for release packaging and quality gates.

## Workflow

1. Inspect the repository:
   - Identify stack from files such as `package.json`, `pnpm-lock.yaml`, `pyproject.toml`, `pom.xml`, `build.gradle`, `Cargo.toml`, `pubspec.yaml`, `tauri.conf.json`, `electron-builder`, `Dockerfile`, or app platform folders.
   - Check existing `.github/workflows/`, release docs, package scripts, test commands, and artifact naming.
   - Read `AGENTS.md` or repo instructions before deciding whether any local command is allowed.

2. Choose workflow intent:
   - `ci.yml`: lint/test/typecheck for branches and pull requests.
   - `build.yml`: build packages and upload artifacts on demand or main branch.
   - `release.yml`: tag-driven release that builds, uploads artifacts, and creates or updates GitHub Releases.
   - `pages.yml`: static site or documentation deployment.
   - Split workflows when release permissions or runtime costs differ from normal CI.

3. Design the workflow:
   - Use `workflow_dispatch` for manual builds and tag triggers for releases.
   - Set explicit `permissions`; default to `contents: read`, raise only where needed, such as `contents: write` for Releases or `pages: write` for Pages.
   - Use pinned major versions for official actions and stable third-party actions; verify exact current versions from official sources when precision matters.
   - Add `concurrency` to prevent duplicate branch builds from racing.
   - Use caching for package managers and build tools, but never cache secrets or generated release credentials.
   - Upload artifacts with clear names that include platform, app name, and version when available.

4. Implement safely:
   - Prefer project-native commands already declared in scripts or build files.
   - Use setup actions for runtimes instead of assuming runner state.
   - Put secrets in GitHub Secrets or OIDC-backed provider auth; never write tokens into workflow YAML.
   - Avoid `pull_request_target` unless the workflow is intentionally hardened against untrusted code.
   - For release workflows, build from the checked-out tag or commit, not an unpinned moving branch.

5. Validate without local building:
   - Check YAML indentation and workflow syntax with available lightweight tools.
   - Run `git diff --check` and inspect changed workflow files.
   - If `gh` is authenticated, use `gh workflow list` or `gh workflow view` after push; use `gh run watch` only after a run exists.
   - Do not claim build success until GitHub Actions completes successfully.

## Professional Defaults

- Add a manual `workflow_dispatch` trigger even when using branch/tag triggers.
- Use `fetch-depth: 0` when tag/version history is needed; otherwise keep checkout shallow.
- Use matrix builds for real multi-platform output, not conditional shell branches inside one job.
- Keep build and release jobs separate when release upload requires broader permissions.
- Include artifact retention appropriate to the project, usually 7-30 days for CI artifacts.
- Prefer official release tooling such as `gh release` or `softprops/action-gh-release` for GitHub Releases.
- Preserve generated artifacts as workflow artifacts even when also attaching them to a release.
- Add clear job names and artifact names so failures are easy to diagnose.

## Common Patterns

Read `references/workflow-patterns.md` when creating or repairing workflows for common stacks:

- Node, pnpm, Vite, Electron, Tauri
- Python packages and CLIs
- Java, Maven, Gradle, Spring Boot
- Android and Flutter APK builds
- Rust and cross-platform binaries
- Docker image build and publish
- GitHub Pages static deployment

## Handoff

When done, report:

- Workflow files changed.
- Which triggers were added.
- Which artifacts or release assets the workflow produces.
- Which secrets, if any, the user must configure in GitHub.
- What was validated locally, and what must be verified by the next GitHub Actions run.
