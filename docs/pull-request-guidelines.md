<img align="left" width="210" hspace="28" src="../assets/pull-request-light.svg#gh-light-mode-only" alt="Pull Request illustration" />
<img align="left" width="210" hspace="28" src="../assets/pull-request-dark.svg#gh-dark-mode-only" alt="Pull Request illustration" />

Pull Request is the only way to make changes to `dev` and `main`. Direct push
to these branches is forbidden at the
[GitHub Rulesets](./branch-strategy.md#rulesets) level.

This page describes the process from creating a PR to merging it, merge
strategies for different branch types, code review requirements, and CI.

<br clear="left" />

## Process

1. Create a [topic branch](./branch-strategy.md#branches) from `dev`
   or from `main` for hotfix
2. Make changes, commit following the
   [commit convention](./commit-convention.md)
3. Publish the branch: `git flow publish` or
   `git push -u origin <branch>`
4. Open a Pull Request into `dev` or into `main` for hotfix
5. Wait for CI to pass – status check `Build`, if enabled
6. Pass code review – recommended for `dev`, 1 approval required for `main`
7. Merge using the correct [merge strategy](#merge-strategies)

After merging, the branch is automatically deleted from remote. Locally, you
need to delete the branch manually:

```bash
git checkout dev
git pull
git branch -d feature/my-feature
```

## Merge strategies

The choice of merge strategy depends on the branch type and the target branch.

| Direction | Method | Why |
|---|---|---|
| `feature/*` → `dev` | Squash | Clean history, one commit per feature |
| `bugfix/*` → `dev` | Merge commit | Preserves the context of the fix |
| `hotfix/*` → `dev` | Merge commit | Preserves the context of the fix |
| `dev` → `main` | Merge commit | Release marker, all commits visible |

**Squash**
- Compresses all branch commits into a single commit
- One PR = one clean commit in `dev`
- Intermediate commits do not end up in the `dev` history
- Original commit history is preserved in the PR on GitHub

**Merge commit**
- Preserves all branch commits and adds a merge commit on top
- For `dev` → `main` it serves as a release marker – the boundary between
  releases is clearly visible

[Protect main](./branch-strategy.md#rulesets) allows only merge commit.
[Protect dev](./branch-strategy.md#rulesets) allows both squash and merge
commit.

> [!IMPORTANT]
> GitHub Rulesets cannot restrict the merge method per source branch. Choosing
> squash for feature is a convention that the PR author follows manually.

## Code review

- PR into `dev` – approval is not required, but recommended
- PR into `main` – **1 approval** from another contributor is required
- On a new push, the existing approval is dismissed and a re-review is needed

## CI

On repositories where the
[Require CI](./branch-strategy.md#rulesets) ruleset is enabled, the status
check `Build` must pass before merging into `dev` or `main`. This is a
separate workflow `ci-build.yml` that runs on every PR and checks that the
project builds successfully.

If CI fails, the merge button is blocked. You need to fix the issue, push the
fix, and wait for it to pass again.

> [!NOTE]
> Require CI is not yet enabled on all repositories. On repositories without
> this ruleset, merge is available right after code review.

## Dependabot

Dependabot creates PRs automatically to update dependencies. These PRs go
through the same process: CI, review, merge. Dependabot has a bypass for the
[branch naming rule](./branch-strategy.md#rulesets), since its branches
`dependabot/*` do not follow git-flow naming.

<p align="center">
  <img src="../assets/pull-request-flow-light.png#gh-light-mode-only" alt="Pull Request flow" width="800" />
  <img src="../assets/pull-request-flow-dark.png#gh-dark-mode-only" alt="Pull Request flow" width="800" />
</p>
