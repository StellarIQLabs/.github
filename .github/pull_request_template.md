<!-- Copy this template for PRs in any StellarIQLabs repo. Org-level template from .github/.github/pull_request_template.md -->

## Summary

<!-- What does this PR change and why? Link PRD.md section: e.g. PRD.md:430, PRD.md:899 -->

Fixes # <!-- issue number -->

## Repo & Scope

- [ ] `stellariq-app` (web / api / sdk / contracts)
- [ ] `stellariq-contract` (indexer / price-engine / analytics / routing / internal-api)
- [ ] `stellariq-infra` (terraform / k8s / docker / monitoring / ci)
- [ ] `.github` (org docs / health files)
- [ ] Cross-repo (list dependent PRs)

## Changes

<!-- Bullet list of user-visible and internal changes -->

## ASCII Impact (if architecture/data-flow changes)

```
before:
  [services] -> [db] -> [api]

after:
  [services] -> [cache] -> [api]
```

## Testing

- [ ] `lint` + `typecheck` + `test` + `build` pass locally
- [ ] Unit tests added
- [ ] Integration / e2e added or updated
- [ ] `terraform fmt` + `terraform validate` (infra only)
- [ ] Manual verification steps:

```
pnpm install && pnpm lint && pnpm typecheck && pnpm test
```

## Docs & Contracts

- [ ] PRD section referenced above
- [ ] `docs/` or README updated if API, pipeline or deployment changed
- [ ] `openapi.json` regenerated (`stellariq-app`) if API changed
- [ ] `DexAdapter` contract preserved (`stellariq-contract`) — `PRD.md:899`

## Security

- [ ] No secrets, `.env`, keys or state files committed
- [ ] Auth / rate-limit / validation considered

## Checklist

- [ ] Conventional Commit title: `feat(scope): ...` / `fix(scope): ...` / `docs: ...`
- [ ] One logical change per commit, no backdated history

## Screenshots / Recordings (web UI)

<!-- before/after if applicable -->

## Notes for Reviewers

<!-- Anything reviewers should focus on -->
