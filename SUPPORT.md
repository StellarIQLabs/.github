# Support

## Getting Help

| Channel | Use for |
|---------|---------|
| **Docs** — [`README.md`](./README.md) + [`docs/`](./docs/) | Architecture, quickstart, API, deployment |
| **GitHub Issues** — `StellarIQLabs/<repo>` with templates | Bugs, feature requests, adapter requests |
| **GitHub Discussions** — `StellarIQLabs/.github` (enable per org) | Questions, ideas, show-and-tell |
| **Security** — [`SECURITY.md`](./SECURITY.md) | Private vulnerability reports only |

Before filing, check:

1.  [`docs/DEVELOPMENT_GUIDE.md`](./docs/DEVELOPMENT_GUIDE.md) — workspace + env issues
2.  [`docs/DEPLOYMENT_GUIDE.md`](./docs/DEPLOYMENT_GUIDE.md) — local compose / Terraform / K8s
3.  [`docs/API_SPEC.md`](./docs/API_SPEC.md) — endpoint contract and tiers

## Filing an Issue — What to Include

* Repo (`stellariq-app` / `stellariq-contract` / `stellariq-infra` / `.github`) + commit SHA
* Node / pnpm / Terraform / Docker versions
* `.env` keys present (never paste values) + error log excerpt
* Steps to reproduce + expected vs actual behavior
* `PRD.md` section if the behavior contradicts spec

Issue templates live in `.github/ISSUE_TEMPLATE/` — they pre-fill this checklist.

## Response Expectations

* Community issues and PRs: maintainer triage within **2–3 business days**.
* Security reports: acknowledgment within **48 hours** per `SECURITY.md`.

## Commercial / Enterprise

Enterprise tier `PRD.md:1193` (SLA, dedicated infra, custom data) — contact via the org profile email. Include workload (protocols, assets, swaps/day, API users) so we can size correctly `PRD.md:1148`.

## Versioning

`main` is active until first tagged `v*`. After MVP, releases follow `vMAJOR.MINOR.PATCH` and are noted in each repo's `CHANGELOG.md`.
