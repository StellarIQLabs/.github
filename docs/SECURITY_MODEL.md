# Security Model

## Invariant: Non-Custodial `PRD.md:129`

```
  User Wallet (holds private keys)
      |
      |  1. request quote  { from, to, amount }
      v
  StellarIQ API (Fastify :4000) -> routing-engine  -> unsigned XDR (no keys)
      |
      |  2. returns { xdr: "AAAA...", bestRoute, impact, fees }
      v
  Wallet  (user reviews + signs)
      |
      |  3. signed Tx -> Stellar Network (submitted by wallet, not StellarIQ)
      v
  Stellar (Soroban + DEX)
```

StellarIQ **never** receives, stores, or logs private keys or seed phrases. `stellariq-contract` holds the on-chain logic; `stellariq-app` only builds unsigned transactions via `@stellar/stellar-sdk`; simulation (`stellariq-infra/services/simulator/`) is pre-submission with no key material.

---

## Trust Boundaries

```
  +-----------+      +----------------+      +-----------+
  |  Wallet   | ---> | stellariq-app  | --->| Stellar  |
  | (trusted) |      |  web + api + sdk |    | Network  |
  +-----------+      +-------+--------+      +-----------+
                             |  internal REST :4110
                             v
                      +----------------+      +-----------+
                     | stellariq-data | --->| Postgres |
                     | indexer etc.   |      | Redis    |
                     +----------------+      | SQS      |
                             ^               +-----------+
                             |  RPC (read-only)
                     +-----------+
                     | Stellar RPC |
                     +-----------+
```

* **Ingress:** TLS via `kubernetes/ingress/` (cert-manager + Let's Encrypt), rate-limit + DDoS guards at the ALB/ingress layer before Fastify.
* **App edge:** Zod validation on every route (`packages/schemas`), Helmet, CORS allowlist (per env), global sanitization.
* **Auth:** `x-api-key` optional -> free tier when absent, `401` on invalid; `POST /v1/keys` guarded by `ADMIN_TOKEN`; per-tier Redis counters emit `x-ratelimit-*` + `Retry-After` on `429`.
* **Data plane:** `DexAdapter` input is untrusted ledger/events — every adapter defensively parses XDR and returns `Swap | null`, discarding unknown events rather than throwing.

---

## Secrets Handling

* **No secrets in git** — enforced by `.gitignore` (`.env`, `*.pem`, `backend.hcl`, `**/terraform.tfstate*`), Husky gitleaks where configured, and CI secret scan.
* **Terraform -> Secrets Manager** — `terraform/modules/secrets` writes `DATABASE_URL`, `REDIS_URL`, `ADMIN_TOKEN`, Soroban passphrases to AWS Secrets Manager.
* **Secrets Manager -> K8s** — `kubernetes/secrets/external-secrets.yaml` (ExternalSecrets operator) syncs into `stellariq-*` namespaces as env. Rotation is a monthly `CronJob` (`kubernetes/secrets/rotation-cronjob.yaml`) plus manual `./scripts/secrets-rotation.sh`.
* **Env at build:** `apps/web` `NEXT_PUBLIC_*` baked at `docker build` — never inject secret `ADMIN_TOKEN` as `NEXT_PUBLIC_*`.

---

## Dependency & Image Security

* `dependabot` + `npm audit` / `cargo audit` in `security-scan.yml` (reused by all repos), failing on high/critical.
* Image scanning (Trivy style) on every `build-images.yml` push.
* Docker bases are pinned digests (`docker/node:22.11.0`, `docker/rust:1.83.0` in `stellariq-infra/docker/`) — rebuilt monthly.
* Supply chain: `pnpm-lock.yaml` / `package-lock.json` / `Cargo.lock` committed, registry cache in CI, no `--force` installs.

---

## Attack Catalogue & Mitigations

| Attack | Mitigation |
|--------|------------|
| Swapped route/lower output via tampered quote | API signs quote payload with route explanations; web shows why winner won; tx builder re-fetches reserves server-side before building XDR |
| Forged swap event poisoning volume/TVL | Adapter `parseSwap` returns `null` on unknown shape; consistency job reconciles TVL vs on-chain (`monitoring/consistency`) and alerts on drift |
| Rate-limit bypass via WS | WS upgrade checks `x-api-key` (`?apiKey=`) and applies same Redis counters + connection caps per IP |
| Postgres injection | Drizzle parameterized queries only, no string-concatenated SQL |
| XSS in dashboard | Next.js escaping, CSP via Helmet, no `dangerouslySetInnerHTML` for chain data (sanitized) |
| Env leakage via logs | Logger redacts `*TOKEN*`, `*SECRET*`, `*KEY*` patterns; Loki retention 30d |
| Supply chain compromise of adapter | Adapter tests run with fixture events in CI; new adapters require fixture + review in `stellariq-data/packages/protocols/` |

---

## Incident Response (link to `SECURITY.md`)

Report privately via `Security -> Report a vulnerability` on any repo. Maintainers triage `<48h`, fix in a private fork, then coordinate a release + advisory + CVE. See `SECURITY.md` for disclosure timeline.
