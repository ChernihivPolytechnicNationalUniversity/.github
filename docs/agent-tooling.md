# Agent Tooling Standard

**Date:** 2026-07-07 · **Status:** Adopted

How Digital University repos expose context and workflows to LLM coding agents
(Claude Code, Codex, Cursor, …). The goal: **agent instructions and skills are
committed team assets in tool-agnostic locations**, with thin per-tool adapters.
Where each doc type lives is defined by the
[documentation IA](./documentation-information-architecture.md); this standard covers
the agent-facing files.

## 1. AGENTS.md — the agent-instructions file

Every active repo carries an `AGENTS.md` at the repo root, following the open
[agents.md](https://agents.md) standard. It is the **only** committed
agent-instructions file — never commit a `CLAUDE.md` (or other tool-branded variant).

Content contract (machine-oriented, terse):

- **Architecture in brief** — 5–10 lines on what the service is and how it fits the
  platform, plus a link to the service's du-docs entry (GitHub URL to the `du-docs`
  source *and* the local sibling-clone path `../du-docs/…`, since the portal is
  auth-walled and agents read the repo source).
- **Commands** — build / run / test / lint, exactly as CI runs them.
- **Conventions & gotchas** — anything an agent would otherwise get wrong.
- **Links** — README, local `docs/` index, repo skills, and this standard.

AGENTS.md may duplicate *facts* from README (commands, ports) but not *prose* —
README stays the human entry point.

## 2. CLAUDE.md — gitignored per-user facade

Claude Code reads `CLAUDE.md`. To keep the repo tool-agnostic, `CLAUDE.md` is a
**gitignored** one-line facade that imports the shared instructions:

```markdown
@AGENTS.md
```

Each developer creates it locally (or lets Claude Code do so). Recommended
`.gitignore` entries:

```gitignore
# per-user agent-tooling files (committed: .claude/skills symlinks)
CLAUDE.md
.claude/*
!.claude/skills
```

## 3. Skills — committed, tool-agnostic layout

Repo-specific agent skills are team assets and live in the repo, following the open
[Agent Skills](https://agentskills.io) standard:

```
.agents/skills/<skill-name>/SKILL.md      ← canonical, committed
.claude/skills/<skill-name>               ← committed SYMLINK → ../../.agents/skills/<skill-name>
```

- `.agents/skills/` is the emerging cross-client convention — Codex and other clients
  scan it natively.
- `.claude/skills/` symlinks give Claude Code native discovery with zero per-user
  setup on macOS/Linux (Claude Code follows symlinks).
- Repo root stays tool-agnostic — the AGENTS.md-over-CLAUDE.md principle applied to
  skills.
- Third-party skills are **never vendored** into repos — install them per-user.

Creating the symlink (macOS/Linux):

```bash
mkdir -p .claude/skills
ln -s ../../.agents/skills/<skill-name> .claude/skills/<skill-name>
git add .claude/skills/<skill-name>
```

### Skill scopes

| Scope | Location | Committed? |
|---|---|---|
| Repo-specific (team asset) | `<repo>/.agents/skills/` + `.claude/skills/` symlinks | yes |
| Workspace/meta | the maintainer's workspace control-center repo | private repo |
| Personal | `~/.claude/skills/` (or `~/.agents/skills/`) | no (per-user) |

## 4. Windows setup (symlink caveat)

Git creates real symlinks on Windows only when both of these hold; otherwise the
committed `.claude/skills/<name>` entries arrive as **plain text files** containing
the link target path (harmless, but Claude Code won't discover the skills):

1. **Developer Mode** is enabled (Settings → System → For developers), which allows
   creating symlinks without elevation, and
2. git has symlinks enabled: `git config --global core.symlinks true` (set **before**
   cloning, or re-checkout afterwards).

Fix an existing clone after enabling both:

```powershell
git config core.symlinks true
git checkout -- .claude/skills
```

**Copy fallback** (no Developer Mode available): delete the text-file placeholders
and copy the skill directories instead, then hide the local change from git:

```powershell
Remove-Item .claude\skills\<skill-name>
Copy-Item -Recurse .agents\skills\<skill-name> .claude\skills\<skill-name>
git update-index --skip-worktree .claude/skills/<skill-name>
```

Note: `.agents/skills/` itself always works regardless — Codex-family tools are
unaffected; the caveat is only Claude Code's `.claude/skills/` discovery path.

## 5. Rollout checklist (per repo)

- [ ] `AGENTS.md` at root per §1, linking this standard.
- [ ] `CLAUDE.md` + `.claude/*` gitignored per §2 (with the `!.claude/skills` exception).
- [ ] Skills (if any) under `.agents/skills/` with `.claude/skills/` symlinks per §3.
- [ ] README `## Documentation` block links this standard (template:
  [`du-gateway/README.md`](https://github.com/ChernihivPolytechnicNationalUniversity/du-gateway/blob/main/README.md)).
