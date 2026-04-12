<img
  align="left"
  width="210"
  hspace="28"
  src="../assets/git-flow-icon.png"
  alt="Gitflow illustration"
/>

This organization uses **Gitflow** a branching model first described by
Vincent Driessen in 2010. Rules are enforced at the GitHub level through
Rulesets and will not let you push a branch with an invalid name or merge a PR
without following the process.

<a href="https://git-flow.sh">
  <img src="https://img.shields.io/badge/git--flow.sh-Read%20the%20Guide-FE5196?style=for-the-badge&logo=git&logoColor=white" alt="git-flow.sh" />
</a>
<a href="https://nvie.com/posts/a-successful-git-branching-model/">
  <img src="https://img.shields.io/badge/Original%20Article-Vincent%20Driessen-076678?style=for-the-badge&logo=git&logoColor=white" alt="Original article" />
</a>

<br clear="left" />

## Branches

Gitflow splits branches into two groups: **base** and **topic**.

Base branches live forever. Changes only get in through a Pull Request.

- **`main`**: production. Every merge here is tagged with a version following
  [SemVer](./versioning.md). Changes come from `dev` (release) or from
  `hotfix/*` (urgent fix).
- **`dev`**: latest delivered changes for the next release. All finished
  features end up here. Default branch on GitHub.

Topic branches are created for a specific task and deleted after completion.

- **`feature/*`**: new functionality. From `dev`, back into `dev`.
- **`bugfix/*`**: a bug fix that has not reached production yet. From `dev`,
  back into `dev`.
- **`hotfix/*`**: an urgent production fix. From `main`, back into `main`,
  with changes pulled into `dev`.
- **`support/*`**: support for an older product version. From a `main` tag of
  the old version.

Difference between `bugfix` and `hotfix`: bugfix – the bug was found during
development, before production. Hotfix – production is already broken and
waiting for the next release is not an option.

## Naming

Branch name: type and name separated by `/`. The name must follow strict
**kebab-case**: lowercase Latin letters, digits, and hyphens.

Allowed:

```text
feature/export-csv
bugfix/broken-pagination
hotfix/crash-on-login
```

Not allowed:

```text
feature/MyFeature       uppercase letters
feature/my_feature      underscores
fix/something           invalid type
feat/azure-ad-sso       invalid type (feat instead of feature)
```

Validation happens on push. Locally git-flow does not check the name, so you
will only find out about a mistake at the moment of `git push`.

## Rulesets

Every repository has five rulesets configured. Bypass is disabled for all
contributors, except for the cases listed below.

**Protect dev**
- PR required
- Merge methods: squash, merge commit
- Protected from deletion and force push

**Protect main**
- PR required, 1 approval needed
- Merge method: merge commit only
- Protected from deletion and force push

**Enforce git-flow naming**
- Checks kebab-case naming for all branches except `main` and `dev`
- Bypass: Dependabot (creates `dependabot/*` branches)

**Protect release tags**
- Only Repository Admin can create `v*` tags

**Require CI**
- PR will not merge until status check `Build` passes
- Not yet enabled on all repositories

> [!NOTE]
> Separately from rulesets, **auto-delete head branches** is enabled at the
> repository level: after a PR is merged, the branch is automatically deleted
> from remote.

## git-flow-next

A CLI tool that automates working with Gitflow. Successor to `git-flow` and
`gitflow-avh`. Not required, but makes creating and publishing branches easier.

### Installation

The example below is for Linux (amd64). For other systems, find the right
binary in [releases](https://github.com/gittower/git-flow-next/releases).

Find the URL of the latest version:

```bash
curl -sL https://api.github.com/repos/gittower/git-flow-next/releases/latest \
  | grep browser_download_url
```

Download and install:

```bash
curl -L -o git-flow-next.tar.gz \
  https://github.com/gittower/git-flow-next/releases/download/v1.1.0/git-flow-next-v1.1.0-linux-amd64.tar.gz
tar xzf git-flow-next.tar.gz
sudo install -m 755 git-flow-v1.1.0-linux-amd64 /usr/local/bin/git-flow
git flow version
```

### Initialization

For repositories in this organization:

```bash
git flow init --preset=classic --defaults --develop=dev
git flow config delete topic release
git flow config edit topic feature --upstream-strategy=squash
```

The first line creates a configuration in `.git/config`: base branches `main`
and `dev`, topic types `feature`, `bugfix`, `hotfix`, `support`. The second
removes release branches which are not used. The third sets the squash strategy
for feature branches so the config matches the
[merge convention](./pull-request-guidelines.md).

### Main commands

Create a branch and switch to it:

```bash
git flow feature start export-csv
git flow bugfix start broken-filter
git flow hotfix start crash-on-login
git flow feature start export-csv --fetch  # pull remote parent before creating
```

Publish to remote for a PR:

```bash
git flow publish
```

Update from parent (pull fresh changes from `dev`):

```bash
git flow feature update
```

> [!TIP]
> After a PR is merged on GitHub, you need to delete the local branch manually:
> ```bash
> git checkout dev
> git pull
> git branch -d feature/export-csv
> ```

Other useful commands:

```bash
git flow feature list                    # all feature branches
git flow feature checkout export-csv     # switch to a branch
git flow feature rename old-name new     # rename
git flow feature delete experiment       # delete a failed experiment
git flow feature delete -r experiment    # delete from remote too
```
