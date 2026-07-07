# Where documentation lives

The full rule set is the [documentation IA](./documentation-information-architecture.md);
the portal mirror lives at **https://docs.npp.whiteforge.ai** → Contributing → "Where
documentation lives".

Quick reference:

| What | Where |
| --- | --- |
| Service build/run/test, links | the service repo's `README.md` (with the standard `## Documentation` block) |
| Agent instructions & repo skills | the service repo's `AGENTS.md` + `.agents/skills/` — per the [agent-tooling standard](./agent-tooling.md) |
| Service internals, deep-dives, diagrams | the service repo's `docs/` |
| Platform architecture (arc42), ADRs, IAM how-tos, component briefs | du-docs (`docs.npp.whiteforge.ai`) |
| Product / end-user docs (Ukrainian) | du-docs product sections |
| Engineering process (branching, commits, PRs, versioning, releases, observability, repo structure, agent tooling) | this repo, `docs/` |
| API contracts / OpenAPI | `du-api-contracts` repo |

Technical docs are English; product/user docs are Ukrainian. One canonical home per
topic — link, don't copy; mark unimplemented designs as proposals.
`digital-university-architecture` is obsolete.
