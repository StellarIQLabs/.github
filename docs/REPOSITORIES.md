# Repositories

## Overview

```
  github.com/StellarIQLabs/
  +-- stellariq-app         product  (web + api + sdk)
  +-- stellariq-data        data     (indexer + price + analytics + routing)
  +-- stellariq-contract    contracts (soroban Rust workspace, standalone)
  +-- stellariq-infra       ops      (terraform + k8s + ci/cd + monitoring + contract deploys)
  +-- .github               health + docs  (this repo)
```

Build order: `infra (net, db)` -> `data (intelligence flows)` -> `contract (on-chain IDs)` -> `app (consumes data + contract IDs)` -> `infra (workloads, ingress, contract deploys)`. Full pipeline diagram in `ARCHITECTURE.md`.

---

## stellariq-app — Product & API `PRD.md:258`

```
stellariq-app/
+-- apps/
|   +-- web/                Next.js 14 App Router, Tailwind (@stellariq/ui preset), lightweight-charts
|   |     app/              (/) overview, /assets[/asset], /markets[/pair], /pools[/pool], /swap
|   |     components/       stat cards, tables, charts, whale feed, swap forms
|   |     .env.example      NEXT_PUBLIC_API_URL, NEXT_PUBLIC_WS_URL, NEXT_PUBLIC_SWAP_ROUTER_ID
|   +-- api/                Fastify 5, Zod, Helmet, CORS, sanitization, Redis rate-limit
|         src/routes/       assets, prices, markets, pools, swaps, quote, routes, analytics, keys
|         src/ws/           gateway for price|trades|liquidity, ping/pong
|         src/auth/         api-key issuance (ADMIN_TOKEN gated) + tier limits
|         src/lib/          contracts.ts (router interfaces), txBuilder.ts (unsigned XDR), wallet.ts
|         .env.example      DATABASE_URL (proxied), REDIS_URL, ADMIN_TOKEN, INTERNAL_DATA_URL
+-- packages/
|   +-- types/              Asset, Market, Pool, Swap, Price, Quote, Route, Candle
|   +-- schemas/            Zod schemas (validation at the edge)
|   +-- ui/                 tokens, primitives, charts, Storybook
|   +-- sdk/                Typed REST + WS client (retry, subscribe helpers)
+-- tests/e2e/              Playwright (boots api + prod web)
+-- .github/workflows/ci.yml   (Node quality/test/build/e2e/images — no Soroban job; contracts live in stellariq-contract)
+-- pnpm-workspace.yaml (apps/*, packages/*), tsconfig.base.json, .eslintrc.cjs, .prettierrc.json, .husky/
```

> `contracts/` was removed from this repo. Soroban source lives in `StellarIQLabs/stellariq-contract`; app only builds unsigned XDR via `@stellar/stellar-sdk`.

| Script | Runs | Notes |
|--------|------|-------|
| `pnpm install` | all | pnpm 9.15.9 via corepack |
| `pnpm dev:api` | api | `tsx watch` -> `http://localhost:4000` |
| `pnpm dev:web` | web | Next dev -> `http://localhost:3000` |
| `pnpm build` | all | `pnpm -r build`, topological |
| `pnpm lint / typecheck / test` | all | recursive gates |
| `pnpm test:e2e` | e2e | Playwright, needs api + prod web |

Env: copy `.env.example` -> `.env` at root, `apps/web/.env.example`, `apps/api/.env.example`. Missing keys fail fast with a clear message. `NEXT_PUBLIC_SWAP_ROUTER_ID` points at IDs deployed from `stellariq-contract`.

---

## stellariq-data — Data & Intelligence `PRD.md:295` (named stellariq-data in PRD)

```
stellariq-data/
+-- apps/
|   +-- indexer/            RPC pollers, ledger+event ingest, checkpoints, backfill
|   |     src/{rpc.ts,poller.ts,index.ts}
|   |     src/services/{assetDiscovery,marketIndexer,poolIndexer,swapIndexer,consistency}
|   +-- price-engine/       vwap.ts, median.ts, outlier.ts -> { price, confidence, timestamp, sources }
|   +-- analytics-engine/   volume, liquidity, pool metrics, OHLCV candles, signals/*
|   +-- routing-engine/     direct.ts, multihop.ts, split.ts, impact.ts, ranking + explanation
|   +-- internal-api/       node:http REST :4110 for app (assets, prices, markets, pools, quotes)
+-- packages/
|   +-- core/               getConfig(), logger, Redis cache, queues (QUEUES.*), key helpers
|   +-- adapters/           ledger/event decoding
|   +-- models/             Drizzle models (asset, market, pool, swap, price, candle)
|   +-- protocols/          stellar-dex/, soroswap/, phoenix/, aquarius/ + registry + AMM math
|         DexAdapter PRD.md:899  { getPools(), getPool(), parseSwap(), getQuote() }
+-- database/
|   +-- client.ts, config.ts, migrate.ts (drizzle-run), drizzle.config.ts
|   +-- migrations/         0001_assets ... 0006_indexes
+-- scripts/backfill.mjs
+-- tests/                  unit/* + integration/pipeline (node:test)
+-- docker-compose.yml      postgres + redis + engine services
+-- .env.example            DATABASE_URL, REDIS_URL, STELLAR_RPC_URL, etc.
+-- tsconfig.build.json     (output mirrors repo tree -> dist/apps/.../src/index.js matches Dockerfiles)
```

| Script | Runs |
|--------|------|
| `npm install && npm run build` | all packages + engines |
| `docker compose up -d postgres redis` | dependencies healthy first |
| `docker compose up -d indexer price-engine analytics-engine routing-engine` | engines |
| `npm run migrate` | drizzle migrator via `database/migrate.ts` |
| `npm test` | `node --import tsx --test` (unit + integration) |

Ports (`stellariq-data/.env.example`): indexer `4101`, price `4102`, analytics `4103`, routing `4104`, internal-api `4110`.

Entry pattern — plain Node:

```
  ts: apps/indexer/src/index.ts  ->  dist/apps/indexer/src/index.js
  run: node dist/apps/indexer/src/index.js   (Dockerfile non-root stellariq user, file-presence healthcheck)
```

---

## stellariq-contract — Soroban Contracts (standalone) `PRD.md:289`

```
stellariq-contract/
+-- contracts/
|   +-- <name>/             Rust Soroban contract (router, example), Cargo.toml, Makefile, src/, tests/
|   +-- example/            hello-world proving cargo test + stellar contract build to WASM
+-- scripts/                build/deploy helpers (called by infra scripts/deploy-contracts.sh)
+-- Cargo.toml / Cargo.lock (workspace)
+-- rust-toolchain.toml     pinned targets + stellar-cli 28
+-- Makefile (optional)     shortcuts for fmt/clippy/test/build
+-- README.md               per-contract build/test/deploy + network IDs
+-- .github/workflows/      Rust fmt/clippy/test/build + stellar-cli
```

| Script | Runs |
|--------|------|
| `cargo fmt --check && cargo clippy -- -D warnings` | lint |
| `cargo test` | unit + integration |
| `stellar contract build --manifest-path contracts/<name>/Cargo.toml` | WASM build |
| `stellar contract deploy ...` (via infra `scripts/deploy-contracts.sh --network testnet\|mainnet`) | deploy |

Env: `STELLAR_RPC_URL`, network passphrase per env (see `stellariq-infra/terraform/soroban.tfvars.example` + `scripts/soroban-networks.sh`). Deployed IDs are wired into `stellariq-app` (`NEXT_PUBLIC_SWAP_ROUTER_ID`) and `stellariq-infra` (`kubernetes/` + secrets).

---

## stellariq-infra — Operations `PRD.md:335`

```
stellariq-infra/
+-- terraform/
|   +-- providers.tf        aws/kubernetes/helm/random + s3 remote-state backend
|   +-- main.tf             networking -> postgres/redis/queues/backups/secrets/registry/cluster
|   +-- variables.tf        region, env, sizing, Soroban RPC/passphrase vars
|   +-- backend.hcl.example, soroban.tfvars.example
|   +-- modules/
|   |     networking/       VPC, public/private/database subnets, NAT, isolated SGs
|   |     postgres/         RDS 16 + param group + RDS Proxy + Secrets Manager
|   |     redis/            ElastiCache
|   |     queues/           SQS + DLQs, FIFO price queue
|   |     backups/          AWS Backup daily plan, KMS, retention, no-recent-backup alarm
|   |     secrets/          Secrets Manager (synced to K8s, incl. contract IDs)
|   |     registry/         ECR per service + S3 WASM bucket (contract artifacts)
|   |     cluster/          EKS 1.30 + managed node group + autoscaler
|   +-- envs/{staging,production,demo}/terraform.tfvars
+-- kubernetes/
|   +-- namespaces/         staging/prod quotas + default-deny network policies
|   +-- web/, api/          deployments + services + HPAs + probes
|   +-- indexer/            stateful (cursor PVC) — data
|   +-- price-engine/, analytics-engine/ (+ CronJobs for OHLCV/volume), routing-engine/ (latency-tuned + HPA) — data
|   +-- ingress/            TLS (cert-manager/Let's Encrypt) + rate-limit/DDoS guards
|   +-- secrets/            ExternalSecrets + monthly rotation CronJob
|   +-- jobs/               db-migrate (pre-rollout), demo-seed
+-- docker/                 pinned bases (node:22.11.0, rust:1.83.0 + stellar-cli)
+-- docker-compose.yml      full parity: postgres, redis, localstack, web, api, 4 data engines
+-- .github/workflows/      build-images.yml, test.yml, security-scan.yml, deploy-staging.yml, deploy-production.yml + reusable shared (consumed by app + data + contract)
+-- scripts/                bootstrap.sh, rollout.sh, rollback.sh, smoke.sh, secrets-rotation.sh, deploy-contracts.sh (deploys stellariq-contract), soroban-networks.sh
+-- services/simulator/     tx simulation (TypeScript/Express)
+-- monitoring/             prometheus/, grafana/ (SLO dashboards), loki/, alerts/, uptime/, consistency/, tracing/ (otel+Sentry)
+-- perf/                   k6: api-p95.js, swaps.js, dashboard.js
+-- docs/                   disaster-recovery.md, promotion.md, launch-readiness.md
+-- .env.example
```

Prereqs: `terraform >=1.6`, `aws-cli v2` (SSO), `kubectl >=1.29`, Docker + Compose.

---

## File Naming & Where to Put Things

* New dashboard page -> `stellariq-app/apps/web/app/<route>/page.tsx` + component in `apps/web/components/`.
* New API endpoint -> `stellariq-app/apps/api/src/routes/<name>.ts` + schema in `packages/schemas` + SDK method in `packages/sdk`.
* New adapter -> `stellariq-data/packages/protocols/<name>/` implementing `DexAdapter` + test fixture + registry entry.
* New Soroban contract -> `stellariq-contract/contracts/<name>/` (Rust) + Makefile + deploy via `stellariq-infra/scripts/deploy-contracts.sh`.
* New K8s workload -> `stellariq-infra/kubernetes/<service>/` + terraform module if it needs cloud resources.
* Org doc -> `.github/docs/<name>.md` (this repo); per-repo detail stays in that repo's `README.md` or `docs/`.

---

## Dependency Graph

```
  stellariq-infra/terraform + docker bases
        |
        v
  stellariq-data/database + packages
        |
        v
  stellariq-data/apps/*  --internal-api-->  stellariq-app/apps/api  -->  stellariq-app/apps/web + sdk
        |                                            |
  stellariq-contract (IDs) --------------------------+
        |                                            |
        +---------- stellariq-infra/k8s + ci + contract deploys <--+
```

Breaking this order (e.g. deploying app before data's internal-api is healthy) makes `/ready` return `503` by design — see `docs/DEPLOYMENT_GUIDE.md`.
