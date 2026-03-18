<div align="center">
  <img
    width="100%"
    src="https://capsule-render.vercel.app/api?type=waving&height=165&section=header&text=Contributing%20Guide&fontSize=48&fontColor=fbf1c7&animation=fadeIn&fontAlignY=50&color=0:076678,50:B57614,100:8F3F71"
    alt="Contributing Guide header"
  />
</div>
<img
  align="left"
  width="210"
  hspace="28"
  src="https://raw.githubusercontent.com/devicons/devicon/master/icons/git/git-original-wordmark.svg"
  alt="Git illustration"
/>

In this organization, contributors are expected to follow the **Conventional Commits**
specification when writing commit messages.

It defines a consistent commit format and supports structured metadata such as scopes,
breaking changes, and issue references. Some of these references can also integrate
directly with GitHub workflows, including issue linking and closing keywords.

<a href="https://www.conventionalcommits.org/en/v1.0.0/">
  <img src="https://img.shields.io/badge/Conventional%20Commits-Read%20the%20Specification-FE5196?style=for-the-badge&logo=git&logoColor=white" alt="Conventional Commits specification" />
</a>

<br clear="left" />

## How we write commit messages

Use this format:

```text
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
````

The first line is the most important part. It should be short, specific, and easy to scan in Git history.

* **type** says what kind of change this is
* **scope** says what part of the project changed
* **description** says what was done
* **body** explains why the change was needed
* **footer** adds structured metadata such as breaking changes, issue references, or co-authors

A body is optional, but it is useful when the title alone does not explain the reason behind the change.

A footer is optional too, but it becomes important when the commit introduces a breaking change or should reference an issue. Conventional Commits allows footer tokens in trailer-like format, and GitHub can also recognize keywords such as `Fixes`, `Closes`, and `Resolves` in commit messages to close issues when the commit is merged into the default branch.

## Commit types we use

Use the type that best describes the actual change.

- `feat`: a new feature
- `fix`: a bug fix
- `docs`: documentation-only changes
- `style`: cosmetic changes that do not affect logic
    - spacing, indentation, commas, formatting by Prettier or ESLint
- `refactor`: structural code changes that do not change behavior
- `test`: adding or fixing tests
- `chore`: a catch-all for changes that do not fit better under another type
    - `.gitignore`, `.editorconfig`, linter configs, license headers
- `perf`: performance improvements without changing external behavior
- `ci`: CI/CD configuration changes
    - `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`
- `revert`: reverting a previous commit
    - common practice is to mention the SHA of the reverted commit
- `build`: changes to the build system or production dependencies
    - `package.json`, `webpack.config.js`, `Makefile`, `Dockerfile`, `gradle`

The Conventional Commits specification only gives special semantic meaning to `feat`, `fix`, and breaking changes. The rest are still valid and useful for keeping history structured and easy to scan.

## Body and footers

The **body** is optional.

Use it when the first line is not enough and you need to explain **why** the change was made.

- it may contain any free-form text
- it may be written as one paragraph, multiple paragraphs, or a short outline
- its main purpose is to answer: **why was this change needed?**

The **footer** is also optional, but it is important when you need structured metadata such as breaking changes, issue references, or commit trailers.

Each footer must start with a token followed by either `: ` or ` #`.

Examples:
- `Token: value`
- `Token #123`

Footer tokens normally use `-` instead of spaces, for example:
- `Co-authored-by:`
- `Acked-by:`

The main exception is:
- `BREAKING CHANGE:`

A footer value may be multi-line. Parsing stops when the next valid footer token starts.

### Footer types we use

- `BREAKING CHANGE:`: marks an incompatible change
    - it can be used with **any** commit type
    - the short form is `!` in the header: `<type>[optional scope]!:`
    - in practice, teams often use both together for clarity

- `Fixes #N`: fixes the bug described in the issue
- `Closes #N`: completes the work described in the issue
    - commonly used when the issue is not strictly a bug
- `Resolves #N`: resolves the problem or discussion described in the issue

These keywords do not just document the relationship to an issue. When GitHub sees them in a commit message or pull request description, they can automatically change the issue state and close it after the commit is merged into the default branch.

If you reference multiple issues, repeat the keyword for each one:

- `Fixes #88, fixes #91`

- `Refs #N`: references an issue without closing it
    - use this when the commit is related to the issue, but does not finish it
    - `Refs` is just a conventional reference token here
      - *You can simply use `#N` to mention an issue in a commit*

- `Co-authored-by:`: marks a commit co-author
    - required format: `Name <email@example.com>`
    - if there are multiple co-authors, repeat `Co-authored-by:` on a new line

- `Acked-by:`: indicates that someone reviewed the change and explicitly
  acknowledged it
    - example: `Tech Lead <lead@example.com>`

## Examples

### Simple commits

```text
feat(rating): add teacher rating summary cards
````

```text
fix(alerts): prevent duplicate classroom notifications
```

```text
docs(contributing): clarify footer conventions
```

```text
style(ui): apply Prettier formatting to dashboard components
```

```text
refactor(search): simplify faculty lookup state handling
```

```text
test(auth): add login form validation tests
```

```text
chore(repo): update .gitignore and .editorconfig
```

```text
perf(statistics): reduce chart re-rendering on filter changes
```

```text
ci(actions): cache npm dependencies in GitHub Actions
```

```text
build(vite): update production build configuration
```

```text
revert: feat(rating): add teacher rating summary cards

Reverts commit a1b2c3d.
```

### Commits with a body

```text
fix(api): handle empty response body

Archived records may return an empty response from the backend.
Without this check, the UI crashes while parsing the payload.
```

### Commits with issue references

```text
feat(public-portal): add faculty comparison view

Closes #188
```

```text
fix(alerts): handle repeated escalation edge case

Fixes #88, #91
```

```text
docs(contributing): describes the standards for making commits

Refs ChernihivPolytechnicNationalUniversity/npp-api#8
Refs: 676104e, a215868
```

### Breaking changes

```text
feat(auth)!: remove legacy password reset flow
```

```text
feat(auth)!: remove legacy password reset flow

BREAKING CHANGE: the legacy reset endpoint is no longer supported
```

### Commits with co-authors and acknowledgements

```text
docs(contributing): expand footer examples

Co-authored-by: Jane Doe <jane@example.com>
```

```text
fix(alerts): correct escalation threshold handling

Co-authored-by: Jane Doe <jane@example.com>
Co-authored-by: John Smith <john@example.com>
Acked-by: Tech Lead <lead@example.com>
```

### Full examples

```text
feat(rating)!: redesign faculty rating calculation

The previous calculation mixed display logic and scoring rules.
This version separates scoring, aggregation, and presentation.

BREAKING CHANGE: rating response fields and totals have changed
Closes #205
Co-authored-by: Jane Doe <jane@example.com>
Acked-by: Product Owner <owner@example.com>
```