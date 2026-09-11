# Roadmap

Source: `PRD.md:1314` (phases) + `PRD.md:1405` (Definition of Done) + `PRD.md:1430` (future).

---

## Phase Plan `PRD.md:1314`

| Phase | Weeks | Title | Outcome | Status |
|-------|-------|-------|---------|--------|
| 1 | 1-2 | **Data Foundation** `PRD.md:1316` | Repo setup, RPC integration, DB schema `PRD.md:942`, asset indexing, event ingestion, first adapter seam | Build |
| 2 | 3-4 | **DeFi Indexing** `PRD.md:1330` | Pools, swaps, markets, all MVP adapters (Stellar DEX, Soroswap, Phoenix, Aquarius), price/volume/liquidity | Build |
| 3 | 5 | **Analytics** `PRD.md:1346` | Dashboard `PRD.md:765`, asset/market/pool pages + charts `PRD.md:798`, historical `PRD.md:498` | Build |
| 4 | 6 | **Swap Intelligence** `PRD.md:1362` | Quote engine `PRD.md:543`, route discovery `PRD.md:570`, impact, fees, net-output ranking `PRD.md:602` | Build |
| 5 | 7 | **Developer Platform** `PRD.md:1376` | Auth, API keys, rate limits, WebSocket `PRD.md:739`, docs, SDK, tier rollout `PRD.md:1164` | Build |
| 6 | 8 | **Launch** `PRD.md:1391` | Perf `PRD.md:1121`, security review, monitoring, docs, demo env, production deploy | Next |
| V1.5 | after MVP | Alerts, wallet analytics, Telegram/Discord, more protocols/assets, expanded WS/SDK | Planned |
| V2 | after MVP | Full execution, smart splitting, portfolio, advanced arb, strategy + institutional API | Planned |
| V3 | after MVP | Agent-ready financial intelligence OS — Data + Intelligence + Execution for autonomous agents `PRD.md:1455` | Vision |

---

## Dependency Graph

```
  Phase1  ->  Phase2  ->  Phase3  ->  Phase4  ->  Phase5  ->  Phase6
  infra/     data/       data        data        app/auth     infra launch
  db, net    adapters    + app web   routing     + SDK        prod (+ contract deploy)
  +contract init
```

`infra` day-0 (network, postgres, redis) unblocks all; `data` must emit `internal-api` before `app`'s `/ready` can go `200`; `contract` IDs must be deployed before swap signing works.

---

## MVP Definition of Done `PRD.md:1405`

A user can, end-to-end:

1.  Open the dashboard.
2.  Search for a supported asset.
3.  View its current price.
4.  View historical price data.
5.  View trading volume.
6.  View liquidity.
7.  Compare available markets.
8.  Inspect pools.
9.  View recent swaps.
10. Enter a swap amount.
11. Receive available routes.
12. Compare expected output and price impact.
13. Generate a transaction for signing (unsigned XDR, non-custodial `PRD.md:617`).
14. Access the same market data through the API (`PRD.md:679`).
15. Receive real-time market updates via WebSocket (`PRD.md:739`).

Verified by `stellariq-infra/scripts/smoke.sh` (hits `/v1/markets`, `/v1/prices/{asset}`, `/v1/quote`, WS) and `stellariq-app/tests/e2e` (Playwright).

---

## 90-Day Targets `PRD.md:1225` (post-launch, not preconditions)

```
5+ protocols, 10k+ assets, 100k+ indexed swaps, 50+ developers, 10+ integrations, 1M+ API requests
```

Primary metric: **number of active applications and developers relying on StellarIQ data** `PRD.md:1208`; supporting: API requests, integrations, indexed swaps, MAU, quote requests, etc. `PRD.md:1211`.

---

## What ships when

* **MVP** — everything through Phase 6 is required before `v0.1.0`.
* **V1.5** adds signals that feed the `PRD.md:641` alert layer and the expanded WS channels.
* **V2** completes the `PRD.md:619` execution path (full aggregator, smart order splitting, limit orders, automation).
* **V3** is the `Stellar DeFi Intelligence OS` `PRD.md:1455`.

---

## How to track

* Issues per phase are labeled `phase-1` .. `phase-6` in each repo.
* Project board: filter by `repo:` + `label:phase-*`.
* Spec changes require a PR against `PRD.md` before code.
