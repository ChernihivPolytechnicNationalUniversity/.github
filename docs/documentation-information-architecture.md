# Digital University — Documentation Information Architecture

**Date:** 2026-07-07 · **Status:** Adopted (supersedes the 2026-06-09 version of this document)

One rule set answering: *what type of information lives where, in what language, who
reads it, how it is cross-linked, and how it stays true.* Every documentation type has
exactly **one canonical home**; every other surface **links** to it — never copies.
This spec exists because doc content historically spread over four kinds of homes
(du-docs portal, per-repo files, `.github` standards, workspace meta files) and drift
breeds wherever the rule is unstated.

## 1. Audiences

| Audience | Entry points (in reading order) | What they need |
|---|---|---|
| Developer — student, colleague, or external contributor (same person, different moments) | du-docs portal (platform picture, component briefs) → repo `README.md` + `docs/` (service work) → `.github` standards (process: branching, commits, PRs, releases) | understand the platform, build/run/change a service, follow the process |
| LLM agent (Claude Code, Codex, Cursor, …) | repo `AGENTS.md` + repo skills (service context, workflows) → du-docs for cross-service/platform context when designing or implementing multi-service tasks (the portal is auth-walled — agents read the local `du-docs` repo source) | machine-oriented service context + the same canonical platform picture humans get |
| End user / university staff | du-docs product sections (Ukrainian) | how to use the products |
| Platform maintainer | workspace meta level (see §2.5) | sync state, gaps, verified snapshot |

## 2. The homes and their contracts

### 2.1 du-docs — the canonical human portal (`docs.npp.whiteforge.ai`)

**Owns:** general project description; platform architecture (arc42 12 sections);
**brief** component descriptions (what each service is, why it exists, how it relates);
overall tech stack; architecture decisions (MADR ADRs); IAM how-tos; product/user docs;
doc-authoring guide (`contributing/`).

**Explicitly does NOT own:** service internals (→ repo `docs/`), build/run/test
commands (→ repo README/AGENTS.md), git/PR/release process (→ this repo), API schemas
(→ du-api-contracts).

**Component-description rule:** for each service, du-docs carries a *brief* entry —
role, boundaries, key relationships — and **back-references** the repo's README and
`docs/` deep-dives for anything deeper. Briefs live as entries in arc42
`05-building-blocks.md` + the service-relationship edge table, each linking the repo —
**no** per-service portal pages (less surface to drift).

**Structure:** 5 Docusaurus instances — `platform` (/docs), `npp`, `student-cab`,
`documents` (products), `contributing`. Build gate: `onBrokenLinks: throw`.

### 2.2 Service repo — the canonical home for everything service-scoped

Required files, each with a defined role:

| File | Audience | Contract |
|---|---|---|
| `README.md` | humans | what the service is (2–3 sentences), build/run/test, links. MUST contain the standard `## Documentation` block (template: [`du-gateway/README.md`](https://github.com/ChernihivPolytechnicNationalUniversity/du-gateway/blob/main/README.md)): portal link + `.github` standards link + `AGENTS.md` link + local `docs/` index if present |
| `AGENTS.md` | LLM agents | open standard ([agents.md](https://agents.md)), never a committed CLAUDE.md. Machine-oriented: architecture-in-brief, commands, conventions, gotchas. MUST link README + `docs/` + skills, and the [agent-tooling standard](./agent-tooling.md) |
| `docs/` (optional) | humans | **service-scoped deep-dives only**: internals, service API detail, business logic, diagrams, guidebooks. Linked from README; back-referenced from the service's du-docs entry |
| `.agents/skills/<name>/SKILL.md` (optional) | LLM agents | repo-specific skills, committed (team asset) — see the [agent-tooling standard](./agent-tooling.md) |
| `.claude/skills` | Claude Code | ONE committed dir-level **symlink** → `../.agents/skills` — see the [agent-tooling standard](./agent-tooling.md), incl. the Windows caveat |
| `CLAUDE.md` | Claude Code (per-user) | **gitignored** thin facade `@AGENTS.md` |

**Ban list for repo docs (this is where split-brain comes from):**

- No platform-level architecture (overall auth flow, other services' behavior, event
  bus design) — link du-docs instead.
- No copies of `.github` standards.
- No aspirational/unimplemented designs presented as current — mark proposals
  explicitly or move them to an ADR.

**AGENTS.md ↔ README division:** README is for humans and stays prose-light;
AGENTS.md is for agents and may duplicate *facts* (commands, ports) but not *prose*.
AGENTS.md keeps a 5–10 line "architecture in brief" + link to the du-docs entry —
agents get in-file orientation without needing the (auth-walled) portal.

### 2.3 `.github` (this repo) — canonical engineering process & org-wide standards

**Owns:** [branch strategy](./branch-strategy.md), [commit convention](./commit-convention.md),
[PR guidelines](./pull-request-guidelines.md), [versioning](./versioning.md),
[release process](./release-process.md), [observability standard](./observability-standard.md),
[repository structure](./repository-structure.md), documentation IA (this document),
and the [**agent-tooling standard**](./agent-tooling.md) (AGENTS.md convention, skills
layout, CLAUDE.md facade, skill scopes). GitHub surfaces this repo org-wide — that's
why standards live here and not in du-docs. du-docs `contributing/` links here, never
restates.

### 2.4 du-api-contracts — canonical API schemas

du-docs `platform/api/intro.md` links it. Repos link it where relevant. Nobody
hand-copies schemas.

### 2.5 Workspace meta level — NOT a documentation home

The maintainer's private control-center repo (`du-framework`) holds only what no
public home should carry: repo-sync state, a pending-work register, scan playbooks,
and workspace-level agent skills. **Rule: the meta level must not duplicate content
that exists in du-docs** — wherever the portal covers an area, the meta level keeps
only pointers plus whatever is genuinely meta (verified SHAs, gap annotations).

### 2.6 Deprecated surfaces

- `digital-university-architecture`: obsolete, to be archived; never linked as canonical.
- `docs/claude-code/` in npp-ui: replaced by the [agent-tooling standard](./agent-tooling.md)
  + `.agents/skills/` layout.

## 3. Language policy

Technical documentation → **English**; product/user documentation → **Ukrainian**.
Existing Ukrainian technical pages migrate **incrementally**: any page substantively
touched is translated in the same change; new technical pages start in English.
Per-repo `docs/` deep-dives count as technical → English on touch.

## 4. Cross-linking rules (the link graph)

Mandatory links (— = link, → = direction):

```
repo README ── portal (du-docs) ── .github standards ── AGENTS.md (local)
repo AGENTS.md ── README ── docs/ ── skills ── .github agent-tooling standard
repo AGENTS.md ──► du-docs platform/architecture (cross-service context for agents;
                   GitHub URL + local sibling-clone path, since the portal is auth-walled)
du-docs component entry ──► repo README + repo docs/ deep-dives   (back-reference)
du-docs contributing ──► .github process docs                      (link, never restate)
.github where-docs-live ──► du-docs portal
```

Rules:

1. Every link goes to the **canonical** home — never to a copy.
2. du-docs → repo links use GitHub URLs (the portal is auth-walled; repos are the open side).
3. New doc page ⇒ the author asks "which home?" per §2; wrong-home content is moved,
   not mirrored.
4. The link graph is verified by the du-docs build (`onBrokenLinks: throw`) and by
   periodic workspace drift checks (§5).

## 5. How this stays true (maintenance)

- Periodic workspace sync scans verify docs against code at `origin/main`; found drift
  is registered and fixed — including §2 contract breaches: ban-list violations,
  missing README blocks, canonical-home breaches.
- The du-docs CI build is the link-integrity gate.
- The `## Documentation` README block and the AGENTS.md template are the rollout
  vehicle for any future convention change.
