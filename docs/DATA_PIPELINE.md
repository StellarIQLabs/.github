# Data Pipeline

How Stellar ledger and Soroban events become the intelligence served by the API — the core moat `PRD.md:1280`.

---

## End-to-End Flow `PRD.md:851`

```
  Stellar Network
     |
     v  RPC / Ledger / Events  (Horizon + Soroban RPC)
  +-----------------+
  | Ingestion       |  pollers + cursors (apps/indexer/src/rpc.ts, poller.ts)
  +-----------------+
     |
     v  raw ledgers + contract events (base64 XDR)
  +-----------------+
  | Protocol Decoders|  XDR -> typed events per protocol
  +-----------------+
     |
     v  decoded PoolCreated, Swap, LiquidityUpdate ...
  +-----------------+
  | Protocol Adapters| DexAdapter PRD.md:899  adapters/{stellar-dex,soroswap,phoenix,aquarius}/
  +-----------------+
     |  getPools()  getPool()  parseSwap()  getQuote()
     v
  +-----------------+
  | Normalized Data |  Asset | Market | Pool | Swap | Price  (packages/models)
  +-----------------+
     |
     v  upsert
  +-----------------+
  | PostgreSQL      |  assets, markets, pools, swaps, prices, candles  database/migrations/ 0001..0006
  | + Redis         |  cache + queues (QUEUES.ledgers, prices, quotes)
  +-----------------+
     |
     +---> Price Engine (VWAP, median, outlier rejection, source weighting, confidence) PRD.md:455
     +---> Analytics Engine (volume, liquidity, pool metrics, OHLCV 1H..1M, signals) PRD.md:485
     +---> Routing Engine (direct, multi-hop, split, impact, net-output ranking) PRD.md:570
     |
     v  publish
  +-----------------+
  | StellarIQ API   |  internal-api :4110  ->  stellariq-app :4000  ->  REST / WS / SDK
  +-----------------+
```

---

## Indexer Internals (`stellariq-contract/apps/indexer`)

### Polling loop

```
  cursor = loadCheckpoint() || genesis
  loop:
    batch = rpc.getLedgers(cursor, limit=100)   // paginated, respects RPC rate limits
    for ledger in batch:
      decoded = decoders.decode(ledger.events)  // per-protocol XDR parsers
      for event in decoded:
        adapter = registry.pick(event.protocol) // DexAdapter dispatch
        swap = adapter.parseSwap(event)         // Swap | null
        if swap:  enqueue(QUEUES.swaps, swap)
        poolUpdate = adapter.getPool(event.poolId) // on PoolCreated/Liquidity
        assetDiscovery(event.assetIds)          // PRD.md:418
      write Postgres in a transaction
      updateCheckpoint(ledger.sequence)
    on error: retry with backoff, DLQ after N, alert via monitoring/alerts
```

* **Checkpoint resume:** Cursor stored in Postgres (`checkpoints` row) — survives pod restarts (`PRD.md:1133` auto retries). `scripts/backfill.mjs` replays any gap.
* **Asset discovery `PRD.md:418`:** New `code:issuer` seen in any pool or swap -> insert `Asset` with `verification_status=unverified`, then enrichment fetches `name`, `home_domain` via Horizon.
* **Consistency check:** `src/services/consistency` compares last indexed pool reserves vs on-chain via adapter `getPool()` hourly; drift metric exported to Prometheus (see `stellariq-infra/monitoring/consistency`).

### Adapter contract

```ts
// PRD.md:899  packages/protocols/<name>/adapter.ts
interface DexAdapter {
  getPools(): Promise<Pool[]>
  getPool(poolId: string): Promise<Pool>     // reserves, TVL, fee
  parseSwap(event: unknown): Swap | null     // null = not a swap event from this protocol
  getQuote(req: QuoteRequest): Promise<Quote> // for routing-engine
}
```

Adding a protocol is four lines `PRD.md:925`:

```
  Create packages/protocols/<name>/  ->  implement DexAdapter  ->  registry.register(name, adapter)  ->  restart indexer
```

Adapters are deliberately isolated; Horizon parsing for `stellar-dex` lives next to Soroban XDR for `soroswap`/`phoenix` with no shared leakage beyond `Swap`/`Pool` types.

Supported at MVP `PRD.md:1011`: Stellar native DEX + Soroswap + Phoenix + one additional major AMM (Aquarius) — architecture already handles `10+ protocols` `PRD.md:1148`.

---

## Price Engine (`apps/price-engine`) `PRD.md:430`

```
  Sources per asset
    +-- Stellar DEX orderbook midpoints
    +-- AMM pool spot prices (per-protocol reserves -> price)
    +-- protocol liquidity weighting
    +-- (future) oracle feeds where appropriate
            |
            +--> VWAP  (volume-weighted, primary V1)  PRD.md:455
            +--> Median (fallback)
            +--> Source weighting + Outlier detection (z-score / IQR filter)
            |
            v  { asset: "XLM", price: 0.2374, currency: "USD", timestamp, sources: 8, confidence: 0.998 } PRD.md:442
            |
            +--> Redis cache key `price:XLM` (TTL ~5s) + Postgres `prices` history (bulk insert)
            |
            +--> internal-api GET /v1/prices/{asset} + /history?timeframe=...
```

Methodology progression: V1 `VWAP + median + weighting + outlier` -> future `TWAP + confidence intervals + liquidity-adjusted` `PRD.md:462`.

Confidence: `1 - (stddev / price)` clamped; downstream API always returns it so callers can gate automated strategies.

---

## Analytics Engine (`apps/analytics-engine`)

```
  Postgres swaps/pools/prices
        |
        +--> Volume  (rolling 24h + buckets per PRD.md:485)
        +--> Liquidity / TVL  (sum reserves * price) + change over 7d  PRD.md:539
        +--> Pool metrics  fee, vol/TVL, trade count, estimated price impact  PRD.md:523
        +--> OHLCV candles  1H 4H 1D 1W 1M built from swaps + price ticks  PRD.md:498
        +--> Signals
             +-- Price discrepancy  market A $0.2368 vs B $0.2394 spread 1.10%  PRD.md:649
             +-- Liquidity event  TVL -22% in 30m  PRD.md:661
             +-- Large swap  $125k XLM->USDC  PRD.md:668  -> whale feed in web overview
```

Signals are *signals*, not guaranteed arb — they simply surface normalized deltas `PRD.md:674`.

---

## Routing Engine (`apps/routing-engine`) `PRD.md:570`

```
  QuoteRequest { from: "XLM", to: "USDC", amount: "10000" }
        |
        +--> Direct routes        XLM -> USDC via each protocol's pools
        +--> Multi-hop routes     XLM -> EURC -> USDC etc.  (BFS depth 2, fan-out capped)
        +--> Split routes         XLM -> USDC split across parallel pools (convex optimization)
        |
        +--> For each candidate route:
             fetch reserves, compute AMM constant-product output, subtract fees + slippage,
             estimate price impact from liquidity  PRD.md:579
        |
        +--> Rank by net output  (NOT lowest fee)  PRD.md:602 -> pick best
        |
        +--> Response { bestRoute, allRoutes: [{ protocol, steps, output, impact, fees, explanation } ...] }
                                      |
                                      +--> explanation string per route ("won because deepest liquidity despite higher fee")
```

Example `PRD.md:584` — route C (split) wins at `2372.03` vs `2367.91` / `2370.42` because net output dominates fee.

Model: constant-product `x*y=k` for AMM pools, orderbook depth for Stellar DEX; `packages/protocols/math` is the shared implementation.

---

## Storage

```
  PostgreSQL (RDS 16, proxied)
  +-- assets           id (code:issuer), code, issuer, name, decimals(7), verification_status, created_at  PRD.md:945
  +-- markets          id, base_asset, quote_asset, protocol, pool_id  PRD.md:958
  +-- pools            id, protocol, token_a, token_b, reserve_a, reserve_b, tvl, fee  PRD.md:967
  +-- swaps            id, tx_hash, protocol, pool, user, input/output asset+amount, timestamp  PRD.md:982
  +-- prices           asset, price, source, timestamp, confidence  PRD.md:995 (+ time-partitioned indexes)
  +-- candles          market, timeframe, open/high/low/close/volume, timestamp
  +-- checkpoints      cursor per poller
  Indexes: swaps(timestamp), prices(asset, timestamp), pools(tvl) with BRIN for scale to 1M swaps/day PRD.md:1151.
```

Backups: `stellariq-infra/terraform/modules/backups` — daily AWS Backup plan + KMS + retention + `no-recent-backup` alarm.

---

## Failure & Recovery

| Scenario | Handling |
|----------|----------|
| RPC throttle | Backoff + `Retry-After`, metrics spike on `rpc_throttled_total` |
| Missed ledger window | DLQ + `backfill.mjs` with `--from <seq> --to <seq>` |
| Price outlier burst | Outlier filter downgrades confidence; API still serves with `confidence <0.9` flag, WS continues |
| Adapter panic (new protocol bug) | Registry quarantines adapter via env flag; pipeline continues for other protocols |
| DB replica lag | `/ready` probes fail -> traffic drains, PITR to last snapshot |

Disaster recovery runbook: `stellariq-infra/docs/disaster-recovery.md`.
