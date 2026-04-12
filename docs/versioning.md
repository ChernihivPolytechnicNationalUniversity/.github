<img align="left" width="210" hspace="28" src="../assets/semver-logo-dark.png#gh-light-mode-only" alt="SemVer illustration" />
<img align="left" width="210" hspace="28" src="../assets/semver-logo-light.png#gh-dark-mode-only" alt="SemVer illustration" />

This organization uses **Semantic Versioning 2.0.0** for versioning releases.

The convention provides a clear git history and straightforward rollback. Each
version corresponds to a specific release and describes the nature of changes
compared to the previous one.

<a href="https://semver.org">
  <img src="https://img.shields.io/badge/SemVer%202.0.0-Read%20the%20Specification-076678?style=for-the-badge&logo=semanticrelease&logoColor=white" alt="SemVer specification" />
</a>

<br clear="left" />

## Version structure

Format: `MAJOR.MINOR.PATCH`. Non-negative integers without leading zeros.

<p align="center">
  <img src="../assets/semver-structure-dark.png#gh-light-mode-only" alt="SemVer structure" width="500" />
  <img src="../assets/semver-structure-light.png#gh-dark-mode-only" alt="SemVer structure" width="500" />
</p>

A version consists of three required parts (version core) and two optional ones
(pre-release, build metadata), which are described in separate sections below.

**PATCH** – a backward compatible bug fix. Behavior only changed where there
was a bug. No new functionality.

| Before | After | What happened |
|---|---|---|
| `1.0.0` | `1.0.1` | fixed a crash on login |
| `1.0.1` | `1.0.2` | fixed table layout on mobile |
| `1.0.2` | `1.0.3` | sidebar was not closing after navigating to another page |

**MINOR** – new backward compatible functionality in the public API, or marking
existing functionality as deprecated. Everything that worked before still works
after. When MINOR is incremented, PATCH resets to 0.

| Before | After | What happened |
|---|---|---|
| `1.0.3` | `1.1.0` | added dark theme |
| `1.1.0` | `1.2.0` | added activity export to CSV |
| `1.2.0` | `1.3.0` | added filters to the public statistics page |

**MAJOR** – backward incompatible changes in the public API. What worked before
may stop working. When MAJOR is incremented, MINOR and PATCH reset to 0.

| Before | After | What happened |
|---|---|---|
| `1.3.0` | `2.0.0` | backend changed the response structure of `/api/auth` |
| `2.0.0` | `3.0.0` | removed SSO login, authentication changed |
| `3.0.0` | `4.0.0` | full UI redesign, navigation and page structure changed |

Other examples: removing an endpoint, changing routing, migrating the state
manager from Redux to Zustand with a new structure.

## Initial development

While MAJOR = 0, the public API is not considered stable. Anything can change
at any time, including breaking changes.

When the version reaches `1.0.0`, it is a promise: the API is stable and
SemVer must be followed strictly.

## Pre-release

A way to mark that a version is not stable. An intermediate state before the
final release. Has lower precedence than the stable version.

Denoted by a hyphen after PATCH and a series of dot-separated identifiers.
Identifiers consist of ASCII alphanumerics and hyphens `[0-9A-Za-z-]`. Numeric
identifiers must not have leading zeros.

Common stages:

- `alpha`: early development, many bugs and things change a lot
- `beta`: core functionality is ready, but there are bugs
- `rc`: ready for release unless critical bugs are found
- Any custom ones: `canary`, `nightly`, `preview`

### Real-world scenario

| Version | What happened |
|---|---|
| `1.0.0-alpha.1` | deployed to a test server, showed to the client |
| `1.0.0-alpha.2` | fixed the form and rating |
| `1.0.0-beta.1` | opened access to more users |
| `1.0.0-beta.2` | optimized loading |
| `1.0.0-rc.1` | final check before release |
| `1.0.0-rc.2` | fixed a bug |
| `1.0.0` | stable release |

## Build metadata

Technical information about a specific build. Does not affect version
precedence. Denoted by a `+` sign after PATCH or pre-release.

| Example | What it means |
|---|---|
| `1.0.0+20240315143000` | build date and time |
| `1.0.0-beta+exp.sha.5114f85` | commit SHA |
| `1.0.0+build.1337` | build number |

> [!NOTE]
> In practice, build metadata is not used in product versions.
