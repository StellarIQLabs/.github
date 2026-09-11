# Glossary

| Term | Meaning | PRD Ref |
|------|---------|---------|
| **StellarIQ** | DeFi intelligence platform for Stellar/Soroban — intelligence layer, not a DEX | `PRD.md:4` |
| **Stellar / Soroban** | L1 + smart contract platform StellarIQ indexes | `PRD.md:9` |
| **Asset** | `code:issuer` (or `XLM` native), plus `name, decimals(7), price, volume24h, liquidity, verification_status` | `PRD.md:392` |
| **Issuer** | Stellar account that issues an asset (null for native `XLM`) | `PRD.md:398` |
| **Market** | `base/quote` pair aggregated across protocols (e.g. `XLM/USDC`) — may have multiple pools | `PRD.md:473` |
| **Pool** | AMM or orderbook liquidity pool with `reserve_a, reserve_b, tvl, fee` | `PRD.md:967` |
| **TVL** | Total Value Locked — `sum(reserves * price)` per pool or market | `PRD.md:1029` expanded |
| **Swap / Trade** | Single `inputAsset -> outputAsset` execution with `txHash, protocol, pool, user, amounts, timestamp` | `PRD.md:982` |
| **OHLCV** | Open/High/Low/Close/Volume candle per timeframe | `PRD.md:494` |
| **VWAP** | Volume-Weighted Average Price — primary V1 pricing method | `PRD.md:455` |
| **Median price** | Median across sources, fallback when VWAP is unstable | `PRD.md:456` |
| **TWAP** | Time-Weighted Average Price — future method | `PRD.md:463` |
| **Source weighting** | Weighting sources by liquidity/depth before aggregation | `PRD.md:457` |
| **Outlier detection** | Filtering anomalous source prices (z-score / IQR) | `PRD.md:458` |
| **Confidence** | `0..1` score (`1 - stddev/price` clamped) returned with every price | `PRD.md:442` |
| **Price impact** | Expected move from reserves / depth at a given amount | `PRD.md:493` / `PRD.md:579` |
| **Slippage** | Tolerance between quoted and executed price | `PRD.md:579` |
| **Route** | Sequence of pools/steps for a swap; direct, multi-hop, or split | `PRD.md:570` |
| **Net output** | `output - fees - slippage` — the ranking invariant `PRD.md:602` | `PRD.md:602` |
| **Multi-hop** | Route via intermediate asset, e.g. `XLM -> EURC -> USDC` | `PRD.md:590` |
| **Split** | Route split across parallel pools for best net output | `PRD.md:595` |
| **DexAdapter** | Interface `{ getPools, getPool, parseSwap, getQuote }` every protocol implements | `PRD.md:899` |
| **Signal** | Derived intelligence — price discrepancy, liquidity event, large swap | `PRD.md:641` |
| **Soroban** | Stellar smart contract runtime (Rust, `stellar-cli`) | `PRD.md:290` |
| **XDR** | Stellar's wire format — StellarIQ builds *unsigned* XDR and returns it | `PRD.md:617` |
| **Non-custodial** | StellarIQ never holds private keys; user/wallet signs | `PRD.md:129` |
| **HPA** | Horizontal Pod Autoscaler (K8s) — auto-scales `api` + `routing-engine` | infra `kubernetes/*/hpa.yaml` |
| **PITR** | Point-in-time recovery (Postgres backups) | `terraform/modules/backups` |

---

## Abbreviations

`AMM` Automated Market Maker · `DEX` Decentralized Exchange · `TVL` Total Value Locked · `XLM` Stellar Lumens · `SLO` Service Level Objective · `RPO/RTO` Recovery Point/Time Objective · `ECR` Elastic Container Registry · `EKS` Elastic Kubernetes Service · `RDS` Relational Database Service.
