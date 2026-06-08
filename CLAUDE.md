# CLAUDE.md

The org-level special `.github` repo for **Digital University** at Chernihiv Polytechnic National University (НУЧП, ChernihivPolytechnicNationalUniversity org). It holds the public organization profile and the shared, org-wide engineering standards / contribution conventions. This is the only PUBLIC repo in the org. Content is mostly Markdown docs and image assets — there is **no code, no build, and no tests**.

## What's here

- `profile/README.md` — the org landing page shown on `github.com/ChernihivPolytechnicNationalUniversity`. Heavy on `shields.io` / `capsule-render` badges; links to stu.cn.ua, the Contributing Guide, and the repo list.
- `CONTRIBUTING.md` — entry point for org-wide contribution standards; an index that badge-links into the `docs/` pages below. Notes most rules are enforced via GitHub Rulesets + CI.
- `docs/branch-strategy.md` — Gitflow branching model; branch names and merges enforced by GitHub Rulesets.
- `docs/commit-convention.md` — Conventional Commits spec for all commit messages.
- `docs/pull-request-guidelines.md` — PR process, merge strategies per branch type, review + CI requirements. PRs are the only way into `dev`/`main`.
- `docs/versioning.md` — Semantic Versioning 2.0.0 for releases.
- `docs/release-process.md` — release flow (currently **empty / stub**).
- `docs/repository-structure.md` — standard layout every org repo follows (`.infra/helm/`, app dir `ui/` or `back/`, env files, CI/CD entry via Dockerfile). Written in **Ukrainian**.
- `docs/observability-standard.md` — `DU-RFC-001`, the normative (RFC 2119) observability standard for services on the shared Kubernetes cluster. Status: Draft.
- `assets/` — PNG/SVG illustrations referenced by the docs (Gitflow, PR flow, SemVer); some use `#gh-light-mode-only` / `#gh-dark-mode-only` for theme-aware rendering.

## How GitHub applies these defaults org-wide

GitHub treats this special `.github` repo as the source of org defaults: `profile/README.md` renders as the public org profile page, and any community-health files here (e.g. `CONTRIBUTING.md`) are **inherited by every org repo that lacks its own copy**. The `docs/` standards are not auto-applied by GitHub — they are conventions linked from `CONTRIBUTING.md` and enforced separately through GitHub **Rulesets** and **CI** in each repo. Note: no issue/PR templates, workflows, `CODEOWNERS`, `CODE_OF_CONDUCT`, `SECURITY`, or `FUNDING` files exist here yet — add them under the GitHub-recognized paths if introducing them.

## Conventions for editing

- These docs are **org-wide standards** — changes affect every Digital University project. Keep them authoritative and consistent.
- Editing this repo follows the very rules it documents: Gitflow branches, Conventional Commits, PRs into `dev`/`main` (no direct push — enforced by Rulesets).
- Preserve the badge/header style (capsule-render banners, shields.io badges) and theme-aware asset suffixes when editing Markdown.
- `observability-standard.md` is normative (RFC 2119 MUST/SHOULD) — edit with that precision; bump its Version field on substantive changes.
- Mixed languages are intentional: profile/standards are largely English; `repository-structure.md` is Ukrainian. Match the existing language of the file you edit.
- Reference paths use absolute GitHub URLs into this repo — keep them valid when moving/renaming files.

No build or test commands — this repo is documentation and configuration only.
