# Architecture

Source of truth for product behavior: `PRD.md` (repo root outside `.github`). This doc explains how the code maps to it.

---

## 1. System Context

```
  +----------------+       REST / WS / SDK         +----------------+
  | Wallets       | <----------------------------> | stellariq-app  |
  | DEX UIs       |                               |  (product)     |
  | Bots / Agents |  unsigned XDR (never keys)    |  web + api + sdk
  | Protocols     | <----------------------------> |  + contracts  |
  +----------------+                               +-------+--------+
                                                         | internal REST (4110)
                                                         v
                                                 +----------------+
                                                 | stellariq-contract
                                                 |  (data/intel)  | <--- Stellar RPC
                                                 |  indexer etc.  | <--- Soroban events
                                                 +----------------+
                                                         |
                                              Postgres + Redis + SQS
                                                         |
                                                 +----------------+
                                                 | stellariq-infra|
                                                 | terraform + k8s|
                                                 +----------------+
                                                         |
                                                    Cloud (AWS)
```

* **Users `PRD.md:143`:** Traders (monitor, compare, swap), Developers (wallets, bots, AI agents), Protocols (analytics), Bots/Agents (structured feeds).
* **Non-custodial invariant `PRD.md:129`:** Private keys never leave the wallet; StellarIQ only builds unsigned XDR and returns it.

---

## 2. Three-Repository Topology `PRD.md:202`

```
  github.com/StellarIQLabs/
  |
  +-- stellariq-app        apps/web (3000) | apps/api (4000) | packages/* | contracts/ | tests/e2e
  +-- stellariq-contract   apps/{indexer,price-engine,analytics-engine,routing-engine,internal-api} | packages/{core,adapters,models,protocols} | database/migrations
  +-- stellariq-infra      terraform/modules/* | kubernetes/* | docker/ | monitoring/ | perf/ | scripts/
  +-- .github              this repo — health files + canonical docs
```

Dependencies (build order): `infra (network, db)` -> `contract (data flows)` -> `app (consumes data)` -> `infra (workloads, ingress, CI/CD)`.

---

## 3. Component Breakdown

### stellariq-app `PRD.md:258`

```
  stellariq-app/
  +-- apps/web               Next.js 14 App Router, Tailwind (preset from @stellariq/ui), lightweight-charts
  |     /          overview  { volume, TVL, trades, top markets/pools, whale feed }
  |     /assets[/asset]      registry + detail
  |     /markets[/pair]      catalog + detail (OHLCV 1H..1M, volume, liquidity, trades)
  |     /pools[/pool]        catalog + detail (reserves, TVL, fees, impact)
  |     /swap                input -> quote -> route compare -> review -> Freighter sign -> confirm
  |     layout, header, sidebar, breadcrumbs, loading/error/not-found
  +-- apps/api               Fastify 5, Zod validation, Helmet, CORS, sanitization, rate-limit (Redis)
  |     REST   /health, /ready, /v1/{assets,prices,markets,pools,swaps,quote,routes,analytics}, /v1/keys, /openapi.json, /docs
  |     WS     /ws  { action: subscribe, channel: "XLM/USDC:price|trades|liquidity" }
  |     tiers  free (anon) -> developer $19-49 -> pro $99-299 -> enterprise  PRD.md:1164
  +-- packages/types         Asset, Market, Pool, Swap, Price, Quote, Route, Candle (shared)
  +-- packages/schemas       Zod schemas for every request/response (validation at the edge)
  +-- packages/ui            Design tokens, primitives, tables, charts, Storybook
  +-- packages/sdk           Typed REST + WS client (retry, typed errors, subscribe helpers)
  +-- contracts/             Soroban workspace (Rust), toolchain pinned, stellar-cli
  +-- tests/e2e              Playwright (boots api + prod web)
```

### stellariq-contract `PRD.md:295` — the intelligence core

```
  stellariq-contract/
  +-- apps/indexer           RPC pollers (rpc.ts, poller.ts), ledger + Soroban event ingest, checkpoints, backfill
  |     services: asset-discovery, market/pool/swap indexing, consistency checks
  +-- apps/price-engine      VWAP (vwap.ts) + median (median.ts) + outlier (outlier.ts) + source weighting -> { price, confidence, timestamp, sources }
  +-- apps/analytics-engine  volume, liquidity, pool metrics (fee, vol/TVL), OHLCV candles, signals (whale, discrepancy, liquidity event)
  +-- apps/routing-engine    direct (direct.ts) + multi-hop (multihop.ts) + split (split.ts), impact/slippage, net-output ranking + explanation
  +-- apps/internal-api      node:http REST (4110) exposing assets/prices/markets/pools/quotes to stellariq-app
  +-- packages/core          getConfig(), logger, Redis cache, queues (QUEUES.*), key helpers
  +-- packages/adapters      ledger/event decoding
  +-- packages/models        Drizzle models for assets, markets, pools, swaps, prices, candles
  +-- packages/protocols     Stellar DEX, Soroswap, Phoenix, Aquarius + registry + AMM math
  |                          DexAdapter { getPools(), getPool(), parseSwap(), getQuote() }  PRD.md:899
  +-- database               postgres client, drizzle-run migrations (0001..0006)
  +-- docker-compose.yml     local parity stack
```

Adapter seam — how a new protocol lands `PRD.md:925`:

```
  Create packages/protocols/<name>/  ->  implement DexAdapter  ->  register in registry  ->  start indexing
```

### stellariq-infra `PRD.md:335`

```
  stellariq-infra/
  +-- terraform/
  |     providers.tf (aws/kubernetes/helm + s3 remote-state), main.tf (wiring), variables.tf, backend.hcl.example
  |     modules/{networking,postgres,redis,queues,backups,secrets,registry,cluster}
  |       postgres -> RDS 16 + param group + RDS Proxy + Secrets Manager
  |       redis    -> ElastiCache
  |       queues   -> SQS + DLQs, FIFO price queue
  |     envs/{staging,production,demo}/terraform.tfvars
  +-- kubernetes/
  |     namespaces/ (staging/prod quotas + default-deny), web/, api/ (HPA), indexer/ (stateful), price-engine/, analytics-engine/ (+ CronJobs), routing-engine/ (latency-tuned + HPA), ingress/ (TLS, rate-limit/DDoS), secrets/ (ExternalSecrets), jobs/ (db-migrate, demo-seed)
  +-- docker/                pinned non-root bases (node 22.11, rust 1.83)
  +-- docker-compose.yml     postgres + redis + localstack(SQS/secrets) + web + api + 4 engines
  +-- .github/workflows/     reusable build/test/security-scan + staging auto-deploy + prod approval + auto-rollback
  +-- scripts/               bootstrap, rollout/rollback, smoke, secrets-rotation, deploy-contracts.sh, soroban-networks.sh
  +-- services/simulator/    tx simulation (pre-submission)
  +-- monitoring/             prometheus, grafana (SLO), loki/promtail, alerts, uptime probes, consistency reconciler, tracing (otel + Sentry)
  +-- perf/                  k6 (API p95 <300ms, quotes <1s, dashboard <2s)
  +-- docs/                  disaster-recovery, promotion, launch-readiness
```

---

## 4. Data Flow & Sequence

### Index to Serve (write path)

```
  Stellar ledger  --->  indexer poller (cursor + checkpoint)  --->  protocol adapters (decode)
        +                                                              |
        v                                                              v
  Soroban events ------------------------------------------------> normalized Pool/Swap/Asset
                                                                      |
                                                                      v
                                             Postgres (assets, markets, pools, swaps, prices)  <-- migrations 0001..0006
                                                                      |
                                              +-----------------------+-----------------------+
                                              |                       |                       |
                                        price-engine            analytics-engine       routing-engine
                                        VWAP/median/...         vol/liq/OHLCV       routes/impact/ranking
                                              |                       |                       |
                                              +-----------------------+-----------------------+
                                                                      |
                                                                  Redis cache
                                                                      |
                                                               internal-api :4110
                                                                      |
                           stellariq-app/api validates (Zod) <--------+
                                      |
                                      +---> REST  /v1/*  +  WS  /ws  +  SDK
                                      |
                                      +---> web dashboard + external consumers
```

### Quote (read path, target <1s `PRD.md:1124`)

```
  user: From XLM To USDC amount 10000
    |
    v  GET /v1/quote?from=XLM&to=USDC&amount=10000
  app/api  --(Zod)-->  routing-engine  (direct + multi-hop + split)
                        |  fetches reserves/TVL from Postgres/Redis
                        |  models price impact / fees / net output  PRD.md:602
                        v
                ranked routes [{ output, steps, impact, fees, explanation } ...]
                        |
                        v  best = max net output
                   { quote, bestRoute, allRoutes }
                        |
    <--------------------+
  { Best Execution: 2370.42 USDC, impact 0.18%, fees 0.30%, review Tx }
```

---

## 5. Deployment Topology (infra)

```
  Internet  ->  ALB/Ingress (TLS via cert-manager, rate-limit + DDoS guards)
                   |
                   +--> k8s/service: web  (HPA, NEXT_PUBLIC_* at build time)
                   +--> k8s/service: api  (HPA, env from ExternalSecrets)
                   |         |--> RDS Postgres 16 (proxied)
                   |         +--> ElastiCache Redis
                   |         +--> SQS (queues)
                   |
                   +--> k8s: indexer (stateful, cursor PVC)
                   +--> k8s: price-engine
                   +--> k8s: analytics-engine (+ OHLCV/volume CronJobs)
                   +--> k8s: routing-engine (latency-tuned nodes + HPA)
                   |
                monitoring: Prometheus scrapes all, Grafana SLO, Loki logs, alerts (incl. no-recent-backup)
                perf: k6 in CI gates p95 <300ms / quotes <1s
                simulator service: pre-submission tx simulation
```

---

## 6. Cross-Cutting Concerns

* **Routing invariant:** Rank by **net output**, not lowest fee `PRD.md:602`. Every route response carries an explanation string so the dashboard can show why route C beat route A.
* **Adapter isolation:** `DexAdapter` is the only contract between protocols and the platform; adding Aquarius or any future AMM never touches indexer core beyond registry.
* **History moat `PRD.md:1280`:** PostgreSQL holds the normalized historical dataset (prices, swaps, liquidity) that powers future signals — time-partitioned indexes on `swaps.timestamp` and `prices.timestamp` carry the `1M swaps/day` headroom `PRD.md:1151`.
* **Availability `PRD.md:1128`:** `99.9%` via readiness probes (`/health`, `/ready`), ingestion retry + consistency checks, backup alarms, auto-rollback on deploy failure.
* **Scale headroom:** Initial sizing already guards `10+ protocols, 100k assets, 1M swaps/day, 10k API users` without redesign; HPA on `api` and `routing-engine` absorbs quote spikes.

---

## 7. Failure Modes

| Failure | Mitigation |
|---------|------------|
| RPC gap / missed ledger | Checkpoint cursor + backfill `scripts/backfill.mjs` + DLQ |
| Price source outlier | Outlier detection + confidence <1, WebSocket still publishes with confidence field |
| Postgres lag | Readiness probe returns 503, load balancer drains, replica failover, PITR |
| Adapter bug | Adapter quarantined via feature flag, registry skips it, rest of pipeline continues |

---

## 8. Where to read next

* `PRD.md:22` (data architecture), `PRD.md:896` (adapter interface), `PRD.md:942` (DB model)
* `docs/DATA_PIPELINE.md` (ingest internals), `docs/API_SPEC.md` (REST+WS contract), `docs/DEPLOYMENT_GUIDE.md` (how to ship)
