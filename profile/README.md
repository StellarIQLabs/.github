# StellarIQLabs

### Intelligence for Stellar DeFi

**StellarIQ** is the data and intelligence infrastructure layer for the Stellar ecosystem — not another DEX, not another explorer, not just another price feed.

```
                MARKET DATA
                      +
                 PRICE DATA
                      +
                 LIQUIDITY
                      +
                 ANALYTICS
                      +
                  ROUTING
                      +
                    API
                      |
                  STELLARIQ  ->  Intelligence layer for wallets, protocols, bots & AI agents
                                   PRD.md:1258
```

**What it does:**

* Indexes Stellar ledger + Soroban events across multiple protocols
* Aggregates prices (VWAP, median, outlier rejection, confidence)
* Tracks pools, liquidity, volume, trades — with history (1H → 1M)
* Ranks swap routes by **net output** across direct / multi-hop / split execution
* Serves everything over a typed REST + WebSocket API and SDK

**Three repos, one platform:**

| Repo | Role |
|------|------|
| [`stellariq-app`](https://github.com/StellarIQLabs/stellariq-app) | Web dashboard, API, SDK, swap UI, Soroban contracts |
| [`stellariq-contract`](https://github.com/StellarIQLabs/stellariq-contract) | Indexer, adapters, price/analytics/routing engines — the intelligence core |
| [`stellariq-infra`](https://github.com/StellarIQLabs/stellariq-infra) | Terraform, K8s, Postgres/Redis, CI/CD, monitoring |
| [`.github`](https://github.com/StellarIQLabs/.github) | Org health files + canonical docs (you are here) |

```
  Stellar Network -> [contract: data] -> [app: product] -> users
                     [infra] underpins both
```

* **Docs:** [`StellarIQLabs/.github/README.md`](https://github.com/StellarIQLabs/.github#readme) is the entry point; `docs/` holds architecture, pipeline, API and deployment guides.
* **Spec:** [`PRD.md`](https://github.com/StellarIQLabs/StellarIQLabs/blob/main/PRD.md) is the source of truth.
* **Contributing:** [`CONTRIBUTING.md`](https://github.com/StellarIQLabs/.github/blob/main/CONTRIBUTING.md) — PRs welcome.

Target without redesign: `10+ protocols, 100k assets, 1M swaps/day, 10k API users` — `PRD.md:1148`.
