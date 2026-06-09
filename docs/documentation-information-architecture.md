# Digital University — Documentation Information Architecture (Design Spec)

**Date:** 2026-06-09
**Status:** Adopted
**Topic:** Where each kind of documentation lives across the DU repos, and how to migrate to it.

## Context

The Digital University (DigitUni / НУЧП) platform spans ~18 active repos plus several
documentation homes that have grown organically and now **overlap**:

- **du-docs** — Docusaurus 3 site, live at `docs.npp.whiteforge.ai` (Entra-gated, internal). Already
  the richest and most current docs: arc42 architecture, MADR ADRs (incl. an accurate BFF+OBO ADR),
  IAM how-tos, a Diátaxis doc-authoring guide, and (stub) product docs.
- **digital-university-architecture** — Fumadocs site (Cloudflare). C4/ERD/sequence diagrams, ADR-001…003,
  OpenAPI copy. **Outdated and now obsolete** — du-docs has superseded it.
- **.github** — org profile + engineering standards (branch strategy, commit convention, PR guidelines,
  versioning, release process, observability, repository structure). GitHub surfaces these org-wide.
- **per-repo** `README.md` + `CLAUDE.md` — service-local docs (created/refreshed in phase 1).
- **du-api-contracts** — OpenAPI 3.1 specs + generated TS/Java clients (the real API source of truth).
- **DIGITAL-UNIVERSITY-OVERVIEW.md** (workspace root) — a verified, English, cross-repo architecture
  overview produced this session (with a refresh recipe in `OVERVIEW-REFRESH.md`).

**Problem:** there is no single rule for *what information belongs where*. Architecture docs and ADRs
exist in two sites; contribution/process guidance is split across three places; OpenAPI is duplicated.
Readers and authors don't know the canonical home, and content drifts out of date (as the architecture
repo did).

**Goal:** define one canonical home per documentation type, with everything else linking to it, and a
phased migration to reach that state. **du-docs is the single human-facing portal.**

## Decisions (confirmed with the user)

1. **du-docs is the single human-facing documentation portal.**
2. **digital-university-architecture is obsolete** — to be archived by the user; not a migration source
   and never referenced as canonical. Its only previously-unique assets (PlantUML diagrams, OpenAPI) are
   superseded by du-docs (diagrams) and du-api-contracts (OpenAPI).
3. **`.github` ↔ du-docs boundary = Approach A:** `.github` stays the canonical home for repo-agnostic
   governance/process (GitHub surfaces it org-wide); du-docs owns all technical & product documentation;
   du-docs links to `.github` for git/commit/PR/release rather than duplicating.
4. **Language by audience:**
   - **Technical documentation → English** (architecture, ADRs, IAM how-tos, API contracts, per-repo
     README/CLAUDE.md, engineering standards, the doc-authoring guide).
   - **User documentation → Ukrainian** (product docs: `npp`, `student-cab`, `documents` — anything an
     end-user / university staff member reads, whether service- or platform-related).
5. **Deliverable of this brainstorm:** this taxonomy spec **plus** the migration plan below.

## The taxonomy — canonical home for every doc type

| Doc type | Canonical home | Language | Notes |
|---|---|---|---|
| Service build/run/test, internals, gotchas | service repo — `README.md` (humans) + `CLAUDE.md` (agents) | English | One service only. README links to du-docs for cross-cutting context. |
| Platform architecture (context, building blocks, runtime, deployment, crosscutting) | **du-docs** `platform/architecture/` (arc42) | English | Single source of truth. `DIGITAL-UNIVERSITY-OVERVIEW.md` content lands here. |
| Architecture decisions (ADRs) | **du-docs** `platform/decisions/` (MADR) | English | Sequential, immutable once accepted; supersede with a new ADR. |
| IAM / identity how-tos | **du-docs** `platform/iam/` | English | Entra app registrations, add-new-service, permissions catalog. |
| Product / user docs | **du-docs** `npp/`, `student-cab/`, `documents/` | **Ukrainian** | Currently stubs — to be grown. |
| Doc-authoring guide (Diátaxis, style, diagrams, tags) | **du-docs** `contributing/` | English | Doc-writing only; carries the Ukrainian style rules used for user docs; links to `.github` for git process. |
| Engineering governance (branch strategy, commit convention, PR guidelines, versioning, release, observability, repo structure) | **.github** `docs/` + `CONTRIBUTING.md` | English | Repo-agnostic; GitHub surfaces org-wide. |
| Org identity | **.github** `profile/README.md` | English (UA acceptable for org landing) | Org landing page. |
| API contracts / OpenAPI | **du-api-contracts** | English | Spec + generated clients. du-docs links/renders an "API reference" page; never copies. |
| Cross-repo architecture overview (English onboarding quick-ref) | workspace `DIGITAL-UNIVERSITY-OVERVIEW.md` | English | Non-canonical convenience mirror; du-docs `platform/architecture/` is canonical. |
| ~~digital-university-architecture~~ | **OBSOLETE** | — | To be archived; not canonical for anything. |

**The one rule:** every documentation type resolves to exactly **one** canonical home. Everything else
*links* to it — no copies.

## De-dupe rules (overlaps to resolve)

- **Architecture & ADRs:** du-docs canonical; architecture-repo versions obsolete.
- **Contributing:** du-docs `contributing/` = doc-writing only; `.github` = git/commit/PR/branch/release.
  Trim du-docs `contributing/pr-workflow.md` (and any commit/branch restating) to *link* to `.github`.
- **OpenAPI:** du-api-contracts canonical; du-docs adds an "API reference" page linking there; the
  architecture-repo copy is obsolete.

## Language policy — consequence to confirm

Adopting "technical = English" means du-docs' **existing** technical sections (`platform/architecture`,
`platform/decisions`, `platform/iam`, `contributing`) — **currently written in Ukrainian** — are slated to
migrate to English. Recommended approach: **incremental** (translate a page when it is next touched / as
part of M2), not a big-bang rewrite. Product docs (`npp`, `student-cab`, `documents`) stay Ukrainian.
No Docusaurus i18n machinery is required — language is determined by content type, and the multi-instance
layout already separates technical from product docs. **Decision:** existing UA technical pages migrate to
English **incrementally** — each page is translated when it is next substantively touched (and new
technical pages are authored in English from the start). No big-bang rewrite.

## Migration plan (Approach A, phased — to execute after this spec is approved)

**M1 — Publish the map.**
- Add a "where docs live" page to du-docs `contributing/` (English) and a 1-screen mirror in `.github`
  (so it is surfaced org-wide). Cross-link them.
- Add a **Documentation** section to each repo's `README.md` linking the portal (du-docs) + standards
  (`.github`). (CLAUDE.md already covers agent context.)

**M2 — Fold architecture into du-docs.**
- Reconcile the verified wiring from `DIGITAL-UNIVERSITY-OVERVIEW.md` into du-docs
  `platform/architecture/`: verify sections 03 (context), 05 (building blocks), 06 (runtime view),
  07 (deployment view) against current reality; update stale parts; add the service-relationship edge
  list and the two diagrams (per du-docs' `contributing/diagrams.md` convention).
- Record known debt in `11-risks-and-tech-debt.md`: studcab-app gateway bypass (to be reworked),
  du-api-contracts published-but-unused, npp-api documented-but-unimplemented `/internal/events/stream`.
- Reduce the workspace `DIGITAL-UNIVERSITY-OVERVIEW.md` to a pointer (or keep as the English onboarding
  mirror, clearly marked non-canonical). Keep `OVERVIEW-REFRESH.md` as the maintenance recipe (retarget
  it at the du-docs architecture pages over time).

**M3 — De-dupe contributing vs standards.**
- Make `.github` the single home for git/commit/PR/branch/versioning/release/observability/repo-structure.
- Trim du-docs `contributing/` duplication to links; keep only doc-writing specifics.

**M4 — API reference.**
- Add a du-docs "API reference" page linking to du-api-contracts (and/or rendering the OpenAPI).
- Stop relying on the architecture-repo OpenAPI copy.

**M5 — Decommission.**
- Add a deprecation pointer to `digital-university-architecture`'s README → du-docs, then the user
  archives the repo.

## Verification

- du-docs `cd ui && npm run build` is green (it throws on broken links / unknown tags — so links and
  tags must be correct).
- No architecture / ADR / OpenAPI duplication remains across homes (grep the obsolete repo is not linked
  as canonical anywhere).
- Every active repo `README.md` has a Documentation section linking portal + standards.
- Spot-check du-docs `platform/architecture/` pages against the verified wiring in
  `DIGITAL-UNIVERSITY-OVERVIEW.md` (gateway routes, event hub, M2M edges, standalone systems).
- The "where docs live" map exists in both du-docs and `.github` and they agree.

## Out of scope

- Writing the actual product (user) documentation content for `npp` / `student-cab` / `documents`
  (separate effort; this spec only fixes *where* it goes and in *what language*).
- Re-authoring all existing UA technical pages in English at once (incremental — see Language policy).
- Any change to `digital-university-architecture` beyond a deprecation pointer (the user archives it).
