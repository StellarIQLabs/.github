# StellarIQ — Intelligence for Stellar DeFi

> **The data and intelligence infrastructure layer for Stellar DeFi.**  
> Real-time prices, liquidity, swaps, markets and routing — unified into one API, dashboard and SDK.

[![Stellar](https://img.shields.io/badge/Network-Stellar%20%7C%20Soroban-000000?logo=stellar)](https://stellar.org)
[![Status](https://img.shields.io/badge/Status-MVP_Build-blue)](./docs/ROADMAP.md)
[![License](https://img.shields.io/badge/License-MIT-green)](./LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](./CONTRIBUTING.md)

## 🚀 Live Demo

| Resource | Link |
| --- | --- |
| **Web dashboard** | https://stellariq-app-web-49hh.vercel.app |
| **API docs (Swagger UI)** | https://stellariq-api-p1hz.onrender.com/docs |
| **API health check** | https://stellariq-api-p1hz.onrender.com/health |
| **Swap router contract (testnet)** | [`CC277AA6…VHSP`](https://stellar.expert/explorer/testnet/contract/CC277AA6E6WZIQRA4N45TQ3O6VV5MUSDMRZCNHO43QENMYXV6E5OVHSP) |

> ℹ️ The API runs on Render's free tier and may take 30–60s to cold-start after idle. Try it live: `curl https://stellariq-api-p1hz.onrender.com/v1/markets`

---

**This repository is the organization health repo for `github.com/StellarIQLabs`.**  
Files here (`CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, templates) apply to all four repos by default. This README is the canonical entry point for the entire project.

---

## Table of Contents

- [What is StellarIQ?](#what-is-stellariq)
- [Why it exists](#why-it-exists)
- [Architecture Overview](#architecture-overview)
- [Repository Map](#repository-map)
- [Core Product Modules](#core-product-modules)
- [Data Pipeline](#data-pipeline)
- [Tech Stack](#tech-stack)
- [Quickstart — All 4 Repos](#quickstart--all-4-repos)
- [API at a Glance](#api-at-a-glance)
- [Non-Functional Targets](#non-functional-targets)
- [Roadmap](#roadmap)
- [Documentation Index](#documentation-index)
- [Contributing & Support](#contributing--support)

---

## What is StellarIQ?

StellarIQ aggregates fragmented Stellar DeFi data into a **normalized, historical, programmable intelligence layer**.

```
 Traditional explorer          StellarIQ
        |                          |
 "What happened?"    =>   "What is happening?" + "Why?" + "What's the best action?"
```

It answers in real time:

* Where is the best price?
* Which pool has deepest liquidity?
* How much price impact will my trade create?
* Where are arbitrage spreads forming?

Tagline: **Intelligence for Stellar DeFi** — `PRD.md:5`  
Network: **Stellar / Soroban** — DeFi Data, Analytics & Swap Intelligence.

---

## Why it exists

Building on Stellar today means rebuilding the same infra repeatedly:

```
  Indexer → Price engine → Pool tracker → Analytics → Historical DB → Routing engine
```

StellarIQ turns that into a reusable platform so wallets, DEX frontends, bots and AI agents can consume intelligence instead of rebuilding it — see `PRD.md:84`.

Principles `PRD.md:114`: **Data first · Developer first · Protocol agnostic · Non-custodial · Modular · Explainable intelligence**.

---

## Architecture Overview

### Three-Layer Model `PRD.md:32`

```
                        StellarIQ
                           |
              +------------+------------+
              |            |            |
              v            v            v
            Data     Intelligence   Execution
              |            |            |
              +------------+------------+
                           v
                    Stellar Network
```

### Four-Repository Deployment `PRD.md:202` (updated)

```
  github.com/StellarIQLabs/
  |
  +-- stellariq-app        Product, API, SDK, Web, Swap  (Next.js + Fastify + SDK)
  +-- stellariq-data       Data & Intelligence (indexer, price, analytics, routing)
  +-- stellariq-contract   Soroban contracts (standalone Rust workspace, deploy scripts)
  +-- stellariq-infra      Cloud, DB, K8s, CI/CD, Monitoring
  +-- .github  <-- you are here (org health + canonical docs)
```

```
                        STELLAR NETWORK  (RPC / Ledger / Events)
                               |
                               v
                     +-------------------+
                     |  stellariq-data   |  <- pollers, decoders, checkpoints
                     |  Protocol Adapters|  <- Stellar DEX, Soroswap, Phoenix, Aquarius
                     |  Price Engine     |  <- VWAP, median, outlier rejection
                     |  Analytics        |  <- volume, liquidity, OHLCV, signals
                     |  Routing          |  <- direct / multi-hop / split
                     +---------+---------+
                               |  API / Events / Cache (Redis)
                               v
                     +-------------------+
                     |   stellariq-app   |
                     |  Web  |  API      |  <- REST + WebSocket
                     |  SDK  |  Swap UI  |  <- typed clients, unsigned Tx builder
                     +-------------------+
                               |  contract IDs / XDR helpers
                               v
                     +-------------------+
                     | stellariq-contract|  <- Soroban source of truth for execution
                     |  example router   |     deployed via infra scripts
                     +-------------------+

                     +-------------------+
                     | stellariq-infra   |
                     |  Terraform | K8s  |
                     |  Postgres, Redis, |  SQS, ECR, Monitoring, CI/CD
                     |  deploys all of the above
                     +-------------------+
                       underpins app + data + contract
```

### How the pieces talk

```
  Stellar RPC  ---> [data: indexer] ---> Postgres (assets, markets, pools, swaps, prices)
                                              |
                                              +-> [price-engine] -> prices + confidence
                                              +-> [analytics-engine] -> OHLCV, volume, liquidity
                                              +-> [routing-engine] -> quotes, routes
                                              |
                                          Redis cache
                                              |
  data: internal-api (port 4110)  <-----------+
          |
          v  (internal REST)
  app: API (Fastify, port 4000) ---> Web (Next.js, port 3000) + External Consumers (SDK / REST / WS)
          |
          +---> contract IDs from stellariq-contract (via env NEXT_PUBLIC_SWAP_ROUTER_ID / contract registry)
          |
          +---> infra: Terraform + K8s deploy data, app, and contract artifacts
```

---

## Repository Map

| Repo | Path | Owns | Key Contents |
|------|------|------|--------------|
| **stellariq-app** | `github.com/StellarIQLabs/stellariq-app` | Dashboard, market/asset/pool pages, swap terminal, REST+WS API, SDK, auth, unsigned Tx building | `apps/web`, `apps/api`, `packages/sdk|types|schemas|ui` (no `contracts/` — moved) |
| **stellariq-data** | `github.com/StellarIQLabs/stellariq-data` | Indexing, adapters, price, analytics, routing — the core intelligence engine | `apps/indexer`, `apps/price-engine`, `apps/analytics-engine`, `apps/routing-engine`, `apps/internal-api`, `packages/core|adapters|models|protocols`, `database/` |
| **stellariq-contract** | `github.com/StellarIQLabs/stellariq-contract` | Soroban contracts, on-chain execution logic, deploy tooling | `contracts/` (Rust workspace), `scripts/`, `Cargo.toml`, `rust-toolchain.toml`, `Makefile` |
| **stellariq-infra** | `github.com/StellarIQLabs/stellariq-infra` | Cloud, DB, containers, K8s, CI/CD, monitoring, contract deploy infra | `terraform/modules/*`, `kubernetes/*`, `docker/`, `monitoring/`, `perf/`, `.github/workflows/` |
| **.github** | `github.com/StellarIQLabs/.github` | Org-wide health files + canonical docs | This repo |

Detailed per-repo maps: [`docs/REPOSITORIES.md`](./docs/REPOSITORIES.md).

---

## Core Product Modules `PRD.md:367`

```
                    StellarIQ
                       |
        +--------------+---------------+
        v              v               v
     Markets        Prices           Pools
        |              |               |
        +--------------+---------------+
                       v
                   Analytics
                       |
                       v
                      Swap  -> Route Optimization -> Swap Execution (quote only in MVP -> unsigned Tx in V1)
```

* **Asset Intelligence `PRD.md:388`** — normalized registry (`code`, `issuer`, `decimals`, price, volume, liquidity, verification).
* **Price Intelligence `PRD.md:430`** — VWAP / median / source weighting / outlier detection -> `{ asset, price, currency, timestamp, sources, confidence }` `PRD.md:442`.
* **Market Analytics `PRD.md:469`** — per-pair price, 24h change, volume, liquidity, spread, OHLCV `PRD.md:494`, market depth.
* **Pool Intelligence `PRD.md:507`** — TVL, reserves, volume, fees, volume/TVL, liquidity change `PRD.md:523`, historical view.
* **Swap & Route Optimization `PRD.md:543` `PRD.md:570`** — evaluates direct, multi-hop, split across pools/protocols; optimizes for **net output** `PRD.md:602`.
* **Market Signals `PRD.md:641`** — price discrepancy `PRD.md:649`, liquidity event `PRD.md:661`, large swap `PRD.md:668`.
* **Contracts `PRD.md:289` (standalone)** — Soroban router / execution contracts now versioned and deployed from `stellariq-contract`; app consumes their IDs/XDR via helpers, infra deploys them.

---

## Data Pipeline `PRD.md:851`

```
  Stellar
     |
     v  RPC / Ledger / Events
  Ingestion (stellariq-data)
     |
     v  Protocol Decoders
  Protocol Adapters   (DexAdapter interface PRD.md:899)
     |   soroswap/  phoenix/  aquarius/  stellar-dex/
     v
  Normalized Data
     |
     v  PostgreSQL (Prices, Pools, Swaps, Markets, Historical) + Redis
  Intelligence Engines
     |   Pricing  Analytics  Routing
     v
  StellarIQ API  (REST + WebSocket)  <- stellariq-app
     |
     v  Dashboard, SDK, External Consumers
     |
     +-> Soroban execution (stellariq-contract IDs referenced in quotes/Tx builder)
```

Protocol add path — `PRD.md:925`:

```
  Create adapter (stellariq-data/packages/protocols/<name>/) -> Implement DexAdapter -> Register -> Start indexing
```

No platform-wide changes required. Contract add path is independent: add Rust contract in `stellariq-contract` -> `stellar contract build` -> infra `scripts/deploy-contracts.sh` deploys.

---

## Tech Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Web | Next.js 14 App Router, Tailwind, `lightweight-charts` | SSR dashboard with shared design system |
| API | Fastify 5 + Zod, WebSocket gateway | Typed, validated edge, consistent envelopes |
| SDK | TypeScript, `zod` | Typed REST + WS for wallets/bots/agents |
| Contracts | Rust, Soroban, `stellar-cli` (in `stellariq-contract`) | Non-custodial tx building `PRD.md:129` |
| Data | Node 20, Postgres 16, Redis 7, SQS, Drizzle (in `stellariq-data`) | Historical dataset moat `PRD.md:1280` |
| Infra | Terraform, EKS 1.30, ECR, RDS, ElastiCache, Helm/K8s | 99.9% availability target `PRD.md:1130` |
| Tooling | pnpm workspaces (app), npm workspaces (data), ESLint/Prettier/Husky, Vitest + `node:test`, Playwright | Monorepo rigor |
| Observability | Prometheus, Grafana, Loki, OpenTelemetry, Sentry | Latency & consistency tracking |

Scale target without redesign `PRD.md:1148`: `10+ protocols, 100k assets, 1M swaps/day, 10k API users`.

---

## Quickstart — All 4 Repos

### Prerequisites

| Tool | Version |
|------|---------|
| Node.js | `>=20` |
| pnpm `9.15.9` (app) / npm `>=10` (data) | `corepack enable` |
| Docker + Compose | recent |
| Terraform `>=1.6`, `aws-cli v2`, `kubectl >=1.29` | infra only |
| Rust stable + `stellar` CLI 28 | `stellariq-contract` only |

### 1. Data layer first — `stellariq-data`

```bash
git clone https://github.com/StellarIQLabs/stellariq-data
cd stellariq-data
cp .env.example .env        # postgres/redis/RPC defaults for localhost
npm install
docker compose up -d postgres redis
npm run build
# drizzle migrate then start: see database/migrate.ts and compose wiring
docker compose up -d indexer price-engine analytics-engine routing-engine
# ports: indexer 4101, price 4102, analytics 4103, routing 4104, internal-api 4110
```

### 2. Contracts — `stellariq-contract` (standalone)

```bash
git clone https://github.com/StellarIQLabs/stellariq-contract
cd stellariq-contract
# Rust toolchain pinned in rust-toolchain.toml
cargo test && stellar contract build --manifest-path Cargo.toml  # or per-contract Makefile
# deploys are driven by stellariq-infra/scripts/deploy-contracts.sh
```

### 3. Product layer — `stellariq-app`

```bash
git clone https://github.com/StellarIQLabs/stellariq-app
cd stellariq-app
pnpm install
cp .env.example .env && cp apps/web/.env.example apps/web/.env && cp apps/api/.env.example apps/api/.env
# NEXT_PUBLIC_SWAP_ROUTER_ID points at contract IDs deployed from stellariq-contract
pnpm dev:api   # http://localhost:4000  ws at /ws
pnpm dev:web   # http://localhost:3000
```

### 4. Infra (full parity locally)

```bash
git clone https://github.com/StellarIQLabs/stellariq-infra
cd stellariq-infra
cp .env.example .env
docker compose up -d          # postgres, redis, localstack, web, api, 4 data engines
# cloud deploy: see docs/DEPLOYMENT_GUIDE.md
```

All services fail fast with clear messages when required env is missing; anonymous API access degrades to free tier, invalid keys get `401`.

---

## API at a Glance `PRD.md:679`

```
GET /v1/assets                 GET /v1/assets/{asset}
GET /v1/prices/{asset}         GET /v1/prices/{asset}/history?timeframe=1H|4H|1D|1W|1M
GET /v1/markets                GET /v1/markets/{pair}
GET /v1/pools                  GET /v1/pools/{pool}
GET /v1/swaps                  GET /v1/swaps/recent
GET /v1/quote?from=XLM&to=USDC&amount=10000
GET /v1/routes
GET /v1/analytics/volume       GET /v1/analytics/liquidity
GET /health  GET /ready  GET /openapi.json  GET /docs  POST /v1/keys (admin-gated)
WS  /ws  -> { action: "subscribe", channel: "XLM/USDC:price" } // also :trades :liquidity
```

Tiers: Free (public dashboard + basic), Developer `$19-49`, Pro `$99-299`, Enterprise custom `PRD.md:1164`. Rate limits via `x-ratelimit-*` + `Retry-After`; `x-api-key` optional (no key = free).

Quotes resolve to contract IDs from `stellariq-contract`; `stellariq-app` then builds unsigned XDR via `@stellar/stellar-sdk` for wallet signing — never custody `PRD.md:129`.

---

## Non-Functional Targets `PRD.md:1117`

| Target | Value |
|--------|-------|
| Cached API | `<300 ms` |
| Quote generation | `<1 s` |
| Dashboard initial load | `<2 s` |
| Availability | `99.9%`, auto retries, health probes |
| Security | No custody `PRD.md:1139`, signed inputs, secret manager, rate limits, simulation |

---

## Roadmap

| Phase | Weeks | Outcome | Docs |
|-------|-------|---------|------|
| 1 Data Foundation | 1-2 | Repo setup, RPC, DB schema, asset indexing, adapter seam (data) + contract workspace init | `PRD.md:1316` |
| 2 DeFi Indexing | 3-4 | Pools, swaps, markets, adapters, price/volume/liquidity | `PRD.md:1330` |
| 3 Analytics | 5 | Dashboard, asset/market/pool pages, historical charts | `PRD.md:1346` |
| 4 Swap Intelligence | 6 | Quote engine, route discovery, impact/fees, ranking | `PRD.md:1362` |
| 5 Developer Platform | 7 | Auth, keys, rate limits, WebSocket, docs, SDK | `PRD.md:1376` |
| 6 Launch | 8 | Perf, security, monitoring, docs, demo, production (incl. contract deploy) | `PRD.md:1391` |
| V1.5 / V2 / V3 | after MVP | Alerts, full execution, smart splitting, portfolio, agent-ready OS | `PRD.md:1430` |

Full roadmap: [`docs/ROADMAP.md`](./docs/ROADMAP.md).

---

## Documentation Index

| Doc | What it covers |
|-----|----------------|
| [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md) | System context, 4-repo topology, data flow, component diagrams |
| [`docs/REPOSITORIES.md`](./docs/REPOSITORIES.md) | Per-repo layout, entry points, build/run/test matrix |
| [`docs/DATA_PIPELINE.md`](./docs/DATA_PIPELINE.md) | Ingest -> normalize -> store -> intelligence -> API with sequence diagram |
| [`docs/API_SPEC.md`](./docs/API_SPEC.md) | REST + WebSocket contract, auth tiers, error envelope, OpenAPI |
| [`docs/DEVELOPMENT_GUIDE.md`](./docs/DEVELOPMENT_GUIDE.md) | Workspace setup, conventions, testing, SDK + contract dev |
| [`docs/DEPLOYMENT_GUIDE.md`](./docs/DEPLOYMENT_GUIDE.md) | Local compose, Terraform, K8s, CI/CD, contract deploy |
| [`docs/SECURITY_MODEL.md`](./docs/SECURITY_MODEL.md) | Threat model, non-custodial execution, secrets, rate limiting |
| [`docs/ROADMAP.md`](./docs/ROADMAP.md) | Phased delivery, Definition of Done, future V1.5-V3 |
| [`docs/GLOSSARY.md`](./docs/GLOSSARY.md) | Asset, market, pool, VWAP, TWAP, price impact, slippage, TVL |

---

## Contributing & Support

* **Contributing** — [`CONTRIBUTING.md`](./CONTRIBUTING.md) (branch, commit, PR, code review).
* **Code of Conduct** — [`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md).
* **Security** — [`SECURITY.md`](./SECURITY.md) — do not open public issues for vulnerabilities.
* **Support** — [`SUPPORT.md`](./SUPPORT.md).
* **PR template** — [`.github/pull_request_template.md`](./.github/pull_request_template.md).
* **Issue templates** — [`.github/ISSUE_TEMPLATE/`](./.github/ISSUE_TEMPLATE/).

License: MIT — see each repo's `LICENSE`.

> Spec is law: [`PRD.md`](../PRD.md) (root outside this repo) is the source of truth for product behavior. When docs diverge, PRD wins.
