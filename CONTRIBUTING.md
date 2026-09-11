# Contributing to StellarIQ

Thanks for contributing — every PR makes the intelligence layer sharper.

## Where to contribute

| Area | Repo | Issues |
|------|------|--------|
| Dashboard, API, SDK | `StellarIQLabs/stellariq-app` | `label: app` |
| Indexer, adapters, pricing, routing | `StellarIQLabs/stellariq-data` | `label: data` |
| Soroban contracts | `StellarIQLabs/stellariq-contract` | `label: contract` |
| Infra, K8s, CI/CD, monitoring | `StellarIQLabs/stellariq-infra` | `label: infra` |
| Org-wide docs & health files | `StellarIQLabs/.github` (this repo) | `label: docs` |

Read `PRD.md` first — product behavior is defined there, not in issues.

## Development setup

See `docs/DEVELOPMENT_GUIDE.md` for per-repo quickstart. Summary:

```bash
# data (intelligence)
git clone https://github.com/StellarIQLabs/stellariq-data && cd stellariq-data
cp .env.example .env && npm install && docker compose up -d postgres redis

# contracts (standalone)
git clone https://github.com/StellarIQLabs/stellariq-contract && cd stellariq-contract
cargo test && stellar contract build --manifest-path Cargo.toml

# app
git clone https://github.com/StellarIQLabs/stellariq-app && cd stellariq-app
pnpm install && cp .env.example .env && pnpm dev:api && pnpm dev:web

# infra local parity
git clone https://github.com/StellarIQLabs/stellariq-infra && cd stellariq-infra
docker compose up -d
```

Prereqs: Node `>=20`, pnpm `9.15.9` (app) / npm `>=10` (data), Docker, Terraform `>=1.6` (infra), Rust + `stellar` CLI `28` (contract repo).

## Branch & commit

* Branch from `main`: `feat/<scope>-<short>` / `fix/<scope>-<short>` / `docs/<short>` / `chore/<short>`.
* Commits: Conventional Commits. Examples:
  * `feat(app): add market detail OHLCV chart`
  * `feat(data): implement phoenix adapter parseSwap`
  * `feat(infra): add price-engine HPA`
  * `docs: update data pipeline diagram`
  * `fix(api): handle 429 retry-after header`
* One logical change per commit. No backdated timestamps — history must be verifiable.

## Pull requests

1.  Open against `main` of the target repo.
2.  Fill the PR template (`.github/pull_request_template.md`).
3.  Link the issue (`Fixes #123`) and the PRD section (`PRD.md:430`).
4.  Ensure CI is green:
    * `lint` + `typecheck` + `test` + `build`
    * infra: `terraform fmt/validate`, `k8s` dry-run where applicable
5.  Request review; address comments with fixup commits, then squash on merge.

### PR checklist (enforced by CI)

* [ ] Tests added or updated
* [ ] Docs updated if API or architecture changed
* [ ] No secrets committed (`.env` files are gitignored)
* [ ] For adapter changes: `DexAdapter` contract preserved `PRD.md:899`
* [ ] For API changes: OpenAPI `openapi.json` regenerated

## Code style

* TypeScript strict. Shared configs live in each repo root (`tsconfig.base.json`, `.eslintrc.cjs`, `.prettierrc.json`).
* Run `pnpm lint && pnpm typecheck && pnpm test` (app) or `npm run ...` (data) or `cargo test` (contract) before pushing.
* Pre-commit: Husky runs `lint-staged` automatically.
* Never commit `.env`, `*.pem`, or Terraform state.

## Adding a protocol adapter

This is the intended extension point `PRD.md:133`:

```
Create adapter -> Implement DexAdapter (getPools, getPool, parseSwap, getQuote) -> Register -> Start indexing
```

Place it in `stellariq-data/packages/protocols/<name>/`, add tests, and wire it in the registry — no core rewrite required. Include a test event fixture.

## Reporting issues

Use the issue templates in `.github/ISSUE_TEMPLATE/` (bug report, feature request, adapter request). Include repo, version, repro steps, and expected vs actual behavior.

## Code of Conduct

By participating you agree to `CODE_OF_CONDUCT.md`.

## License

Contributions are MIT-licensed — same as each repo's `LICENSE`.
