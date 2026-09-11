# API Specification

Source of truth: `PRD.md:679` (REST) + `PRD.md:739` (WebSocket). OpenAPI is served live as `GET /openapi.json` and browsable at `GET /docs`.

---

## Base & Conventions

* **Base URL:** `https://api.stellariq.xyz` (production) / `http://localhost:4000` (dev).
* **Content type:** `application/json` only.
* **Envelopes:**
  * Success: body as documented per endpoint.
  * Error: `{ "error": "CODE", "message": "human text", "statusCode": 429 }` — consistent across REST and WS errors.
* **Validation:** Every request body/query is Zod-validated at the edge (`packages/schemas` in `stellariq-app`). Invalid -> `400` with field-level detail.
* **CORS:** Allowlist per env, preflight handled. Helmet + global sanitization in `apps/api`.
* **Health:** `GET /health` (liveness, always `200` when process up), `GET /ready` (`200` only when downstream `contract: internal-api` is reachable, else `503`).

```
  Client  ->  GET /v1/markets  ->  Fastify  ->  Zod validate  ->  rate-limit (Redis)  ->  auth (optional x-api-key)
                                              ->  handler -> internal-api:4110 / Redis cache -> response
```

---

## Authentication & Tiers `PRD.md:1164`

`x-api-key` header is **optional**. No key = free tier. Invalid key -> `401`.

```
  POST /v1/keys   (guarded by ADMIN_TOKEN env; disabled without it)
  -> { "key": "sk_...", "tier": "developer" }
```

| Tier | Price | Use | Limits (req/min, per tier header) |
|------|-------|-----|-----------------------------------|
| Free | free | Public dashboard, basic data, limited history | low (default when no key) |
| Developer | $19-49/mo | Higher limits, history API, WebSockets | medium |
| Pro | $99-299/mo | High limits, alerts, advanced analytics, signals | high |
| Enterprise | custom | Dedicated infra, SLA, custom data | negotiated |

Headers on every response: `x-ratelimit-limit`, `x-ratelimit-remaining`, `x-ratelimit-reset`, plus `Retry-After` on `429`.

---

## REST Endpoints `PRD.md:679`

### Assets

```
GET /v1/assets?search=XLM&verified=true&page=1&pageSize=25
-> { "assets": [ { "id": "XLM", "code": "XLM", "issuer": null, "name": "Stellar Lumens", "decimals": 7,
                    "price": 0.2374, "volume24h": 4200000, "liquidity": 8700000 } ], "page": 1, "total": 1 }

GET /v1/assets/{asset}   // asset = "XLM" or "USDC:issuer..."
-> { "id": "...", "code": "USDC", "issuer": "G...Issuer", "decimals": 7, "price": 1.0, "volume24h": ..., "liquidity": ..., "markets": ["XLM/USDC", ...] }
```

`PRD.md:388` shape.

### Prices

```
GET /v1/prices/{asset}
-> { "asset": "XLM", "price": 0.2374, "currency": "USD", "timestamp": 1789060000, "sources": 8, "confidence": 0.998 }  PRD.md:442

GET /v1/prices/{asset}/history?timeframe=1H|4H|1D|1W|1M&from=...&to=...
-> { "asset": "XLM", "timeframe": "1D", "candles": [ { "t": 1789000000, "o": 0.235, "h": 0.24, "l": 0.232, "c": 0.237, "v": 4200000 } ] }
```

Methodology V1: VWAP + median + source weighting + outlier detection `PRD.md:455`.

### Markets

```
GET /v1/markets?protocol=soroswap&sort=volume_desc&page=1
-> { "markets": [ { "id": "XLM/USDC", "base": "XLM", "quote": "USDC", "price": 0.2374, "change24h": 0.0214,
                    "volume24h": 4200000, "liquidity": 8700000, "trades": 18294, "spread": 0.0011, "protocol": "soroswap" } ] }

GET /v1/markets/{pair}   // "XLM/USDC" URL-encoded as XLM%2FUSDC
-> full market with OHLCV + market-depth where available + per-protocol breakdown (see PRD.md:473 metrics)
```

### Pools

```
GET /v1/pools?protocol=phoenix&asset=XLM
-> { "pools": [ { "id": "pool-a", "protocol": "phoenix", "tokenA": "XLM", "tokenB": "USDC",
                  "reserveA": "42000000000", "reserveB": "10000000000", "tvl": 4200000, "fee": 300 } ] }

GET /v1/pools/{pool}
-> { "id": "pool-a", "protocol": "phoenix", "tokenA": "XLM", "tokenB": "USDC", "reserves": {...}, "tvl": ..., "volume24h": ..., "fees": ..., "volOverTvl": 0.12, "tradeCount": 183, "priceImpact": 0.0018,
     "history": { "liquidity7dChange": 0.34 } }  // PRD.md:523
```

### Swaps

```
GET /v1/swaps?asset=XLM&protocol=soroswap&page=1
GET /v1/swaps/recent?limit=25
-> { "swaps": [ { "id": "...", "txHash": "abc...", "protocol": "soroswap", "pool": "pool-a",
                  "user": "G...USER", "inputAsset": "XLM", "outputAsset": "USDC",
                  "inputAmount": "10000", "outputAmount": "2370", "timestamp": 1789060000 } ] }
```

### Quotes & Routes `PRD.md:543` `PRD.md:570` — core differentiator

```
GET /v1/quote?from=XLM&to=USDC&amount=10000
-> {
     "from": "XLM", "to": "USDC", "amount": "10000",
     "bestOutput": "2372.03", "bestRouteId": "route-c",
     "priceImpact": 0.0018, "fees": { "network": "0.00001", "protocol": "0.003" },
     "explanation": "Split across pools maximizes net output despite higher fee"
   }

GET /v1/routes?from=XLM&to=USDC&amount=10000
-> {
     "routes": [
       { "id": "route-a", "steps": ["XLM->USDC"], "pools": ["pool-a"], "output": "2367.91", "impact": 0.0019, "fees": 0.003, "explanation": "..." },
       { "id": "route-b", "steps": ["XLM->EURC", "EURC->USDC"], "pools": ["pool-x", "pool-y"], "output": "2370.42", "impact": 0.0015, "fees": 0.006, "explanation": "..." },
       { "id": "route-c", "steps": ["XLM->USDC (split)"], "pools": ["pool-a", "pool-b"], "output": "2372.03", "impact": 0.0018, "fees": 0.003, "explanation": "Split wins on net output" }
     ],
     "rankedBy": "netOutput"  // invariant PRD.md:602
   }
```

Ranking is by **net output**, not lowest fee. Swap execution is quote-only in MVP `PRD.md:609`; V1 returns an unsigned XDR transaction built from the selected route `PRD.md:617`:

```
  GET /v1/quote -> user picks route -> POST /v1/tx/build { routeId } -> { "xdr": "AAAA..." } -> wallet signs -> submit to Stellar
```
Private keys never touch StellarIQ `PRD.md:129`.

### Analytics

```
GET /v1/analytics/volume?from=...&to=...&groupBy=market|protocol|day
GET /v1/analytics/liquidity?from=...&to=...
-> { "series": [ { "t": 1789000000, "volume": 12800000 }, ... ] }
```

### Spec & Keys

```
GET /openapi.json      -> OpenAPI 3.0 (generated from zod schemas)
GET /docs              -> Swagger UI (offline bundle, no external CDN)
POST /v1/keys          -> { "tier": "developer" } body, requires header x-admin-token: <ADMIN_TOKEN>
```

---

## WebSocket `PRD.md:739`

* **URL:** `ws://localhost:4000/ws` (dev) / `wss://api.stellariq.xyz/ws`.
* **Subprotocol:** plain JSON messages.

```
Client ->  { "action": "subscribe", "channel": "XLM/USDC:price" }
Server ->  { "type": "subscribed", "channel": "XLM/USDC:price" }
... then
Server ->  { "type": "price_update", "pair": "XLM/USDC", "price": 0.2374, "timestamp": 1789060000 } PRD.md:752
Client ->  { "action": "subscribe", "channel": "XLM/USDC:trades" }
Server ->  { "type": "trade_update", "pair": "XLM/USDC", "swap": { ... } }
Client ->  { "action": "subscribe", "channel": "XLM/USDC:liquidity" }
Server ->  { "type": "liquidity_update", "pair": "XLM/USDC", "liquidity": 8700000, "change": -0.22 }
Client ->  { "action": "unsubscribe", "channel": "XLM/USDC:price" }
Server ->  { "type": "unsubscribed", "channel": "XLM/USDC:price" }
Client ->  { "action": "ping" }  ->  Server { "type": "pong" }
```

Also handles WebSocket protocol `ping`/`pong` frames. Channels are scoped to `pair:kind`; subscribing twice to the same channel is idempotent. Rate limits apply per WS connection and per `x-api-key` if provided on upgrade (`?apiKey=sk_...` or header on handshake where supported).

---

## Errors & Rate Limits

```
  400 { error: "VALIDATION_ERROR", message: "from is required", statusCode: 400 }
  401 { error: "UNAUTHORIZED", message: "invalid api key", statusCode: 401 }
  404 { error: "NOT_FOUND", message: "asset XLM2 not found", statusCode: 404 }
  429 { error: "RATE_LIMITED", message: "tier limit exceeded", statusCode: 429, headers: { Retry-After: "12", x-ratelimit-* } }
  503 { error: "NOT_READY", message: "data layer unreachable", statusCode: 503 }  // when internal-api is down
  5xx { error: "INTERNAL", message: "...", statusCode: 500 }
```

---

## SDK (`packages/sdk` in stellariq-app)

```ts
import { createClient } from "@stellariq/sdk"
const cli = createClient({ baseUrl: "https://api.stellariq.xyz", apiKey: "sk_..." })
await cli.assets.list({ search: "XLM" })
await cli.prices.get("XLM")
const ws = cli.ws()           // typed subscribe helpers
ws.subscribe("XLM/USDC:price", (evt) => console.log(evt.price))
```

SDK is generated against `openapi.json` so it never drifts from the server contract.
