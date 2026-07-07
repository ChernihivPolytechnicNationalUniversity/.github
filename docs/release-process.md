This organization releases with **Gitflow + SemVer + tag-triggered deploys**: work
accumulates on `dev`, a release PR lands it on `main`, and a `v*.*.*` tag on `main` is
what actually ships to production. The shared CI/CD machinery lives in
[`shared-actions`](https://github.com/ChernihivPolytechnicNationalUniversity/shared-actions).

Prerequisites: [branch strategy](./branch-strategy.md) (Gitflow, rulesets),
[versioning](./versioning.md) (SemVer), [commit convention](./commit-convention.md),
[PR guidelines](./pull-request-guidelines.md).

## The release flow

```text
feature/* ──PR──► dev ──release PR──► main ──tag vX.Y.Z──► production deploy
                   │                                            ▲
                   └─ push to dev auto-deploys the dev env      │
                                         hotfix/* ──PR──► main ─┘ (then back-merge to dev)
```

1. **Develop.** `feature/*` / `bugfix/*` branches merge into `dev` via PR
   (squash or merge commit). Every push to `dev` auto-deploys the service's
   **dev environment** (the repo's `deploy-dev.yml` → dev overlay values, `*-dev`
   Helm release/namespace).
2. **Release PR.** When `dev` holds a shippable increment, open a PR
   `dev → main`. `main` requires 1 approval and merges by **merge commit only**
   (ruleset-enforced). The PR description summarizes the increment — it becomes the
   de-facto release note.
3. **Choose the version.** Per [SemVer](./versioning.md): PATCH = fixes only,
   MINOR = new backward-compatible functionality, MAJOR = breaking change.
   Pre-releases (`-alpha.N` / `-beta.N` / `-rc.N`) are allowed for staged rollouts.
4. **Tag = release.** A repository **admin** creates and pushes the tag (the
   "Protect release tags" ruleset restricts `v*` tags to admins):

   ```bash
   git checkout main && git pull
   git tag vX.Y.Z
   git push origin vX.Y.Z
   ```

5. **The tag deploys.** Pushing `v*.*.*` triggers the repo's prod deploy workflow,
   which calls the reusable
   [`shared-actions` `deploy.yml`](https://github.com/ChernihivPolytechnicNationalUniversity/shared-actions/blob/main/.github/workflows/deploy.yml):
   Docker build → push to GHCR → Azure-OIDC login to the k3s cluster →
   `helm upgrade --install` with the prod overlay values → deploy notification via the
   `notify` action. Watch the Actions run to completion before calling the release
   done.

## Hotfixes

Production broken, can't wait for the next release: branch `hotfix/*` **from `main`**,
PR back into `main`, tag a **PATCH** release (step 4 above), then bring the fix back
into `dev` (merge `main` into `dev`, or cherry-pick) so the next release doesn't
regress it.

## What deploys when — repo variations

| Repo kind | Dev deploy | Prod deploy |
|---|---|---|
| Services (npp-api, du-notifications, npp-ui, digital-documents-api, nuchp-system, …) | push to `dev` | push tag `v*.*.*` |
| Docs sites (du-docs) | — | push to `main` (no tags; a docs "release" is the merge itself) |
| npp-infra (cluster config) | applied per change | applied per change — no versioned releases |

Check the repo's `.github/workflows/` when in doubt — the trigger blocks are the
source of truth.

## shared-actions promotion

All repos consume `shared-actions` **`@main`** — composite actions (`notify`,
`docker-build-push`, `k8s-azure-login`, `collect-commits`) and the reusable
`ci-build.yml` / `deploy.yml` workflows. Merging to `shared-actions` `main` therefore
**is** the release: the change reaches every consumer's next workflow run immediately,
with no version pinning in between.

Consequences:

- A `shared-actions` change is effectively an org-wide CI/CD change — review it with
  that blast radius in mind, and keep it backward-compatible (new inputs optional,
  defaults preserved).
- After merging, verify the first downstream workflow run (any repo's CI or deploy)
  actually passes before walking away.
- If a breaking change is unavoidable, coordinate: land consumer-side workflow updates
  in the same window, or introduce the new behavior behind a new optional input.

## Release checklist

- [ ] `dev` is green (CI passing) and the deployed dev env looks sane.
- [ ] Release PR `dev → main` approved and merged (merge commit).
- [ ] Version chosen per [SemVer](./versioning.md); breaking changes ⇒ MAJOR, called
      out in the PR description.
- [ ] Admin pushed the `vX.Y.Z` tag; prod deploy workflow green end-to-end.
- [ ] For hotfixes: fix back-merged into `dev`.
