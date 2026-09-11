# Development Guide

## Workspace Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Node.js | `>=20` | `node -v` |
| pnpm `9.15.9` | `packageManager: pnpm@9.15.9` | `corepack enable` or `npm i -g pnpm@9.15.9` |
| npm | `>=10` | only for `stellariq-contract` (npm workspaces) |
| Docker + Compose | recent | local parity |
| `terraform >=1.6`, `aws-cli v2`, `kubectl >=1.29` | infra only | |
| Rust stable + `stellar` CLI `28` | `stellariq-app/contracts` only | `rust-toolchain.toml` pins targets |

---

## First Clone — Three Repos

```bash
mkdir StellarIQLabs && cd StellarIQLabs
git clone https://github.com/StellarIQLabs/stellariq-contract
git clone https://github.com/StellarIQLabs/stellariq-app
git clone https://github.com/StellarIQLabs/stellariq-infra
git clone https://github.com/StellarIQLabs/.github
cat .github/README.md   # start here
cat PRD.md               # spec is law (parent of all 3 repos)
```

---

## stellariq-contract — Data Layer

```bash
cd stellariq-contract
cp .env.example .env   # STELLAR_RPC_URL, DATABASE_URL, REDIS_URL, ports 4101..4110
npm install
docker compose up -d postgres redis   # wait healthy: docker compose ps
npm run build
npm run migrate         # drizzle-run via database/migrate.ts (+ drizzle.config.ts)
docker compose up -d indexer price-engine analytics-engine routing-engine
# logs: docker compose logs -f indexer
# health: curl http://localhost:4101/health  (4101 indexer, 4102 price, 4103 analytics, 4104 routing, 4110 internal-api)
npm test                # node --import tsx --test (unit + integration/pipeline)
```

Structure `docs/REPOSITORIES.md` and pipeline `docs/DATA_PIPELINE.md`.

Key conventions:

* Relative cross-package imports: `../../../packages/core/src/getConfig.ts` — no workspace alias magic so `dist/` mirrors repo tree and Dockerfiles (`node dist/apps/<name>/src/index.js`) stay trivial — see `tsconfig.build.json`.
* `packages/protocols/<name>/` implements `DexAdapter { getPools(), getPool(), parseSwap(), getQuote() }` `PRD.md:899`.
* Migrations are `drizzle` sequential (`0001_assets` .. `0006_indexes`); never edit a committed migration — add a new one.
* `scripts/backfill.mjs --from 1000 --to 2000` replays a ledger window after a cursor gap.

---

## stellariq-app — Product Layer

```bash
cd stellariq-app
cp .env.example .env          # DATABASE_URL, REDIS_URL, INTERNAL_DATA_URL=http://localhost:4110
cp apps/api/.env.example apps/api/.env
cp apps/web/.env.example apps/web/.env   # NEXT_PUBLIC_API_URL=http://localhost:4000  NEXT_PUBLIC_WS_URL=ws://localhost:4000/ws
pnpm install
pnpm dev:api    # Fastify on :4000, ws at /ws  (reads data from contract internal-api or mock)
pnpm dev:web    # Next.js on :3000
# or: pnpm dev  (concurrent)
pnpm lint && pnpm typecheck && pnpm test && pnpm build   # gate, same as CI
pnpm test:e2e   # Playwright in tests/e2e (boots api + prod web automatically, see playwright.config.ts)
```

Conventions:

* Zod everywhere — `packages/schemas` defines every request/response; `apps/api/src/routes/*` validates at the edge and returns `{ error, message, statusCode }` envelopes.
* Shared types in `packages/types`; UI tokens in `packages/ui` (Tailwind preset) — `apps/web` imports both.
* SDK: `packages/sdk` typed client wrapping `openapi.json` (served by `apps/api` as `GET /openapi.json` + `GET /docs`). Never hand-roll fetch — update the spec, regenerate.
* Contracts: `cd contracts && cargo fmt && cargo clippy && cargo test`; `pnpm --filter stellariq-contracts toolchain` checks `rust-toolchain.toml`.
* Web routes: `app/(group)/page.tsx` with `loading.tsx`/`error.tsx`/`not-found.tsx` per PRD pages `PRD.md:767`.

---

## stellariq-infra — Local Parity + Cloud

```bash
cd stellariq-infra
cp .env.example .env          # AWS_PROFILE, TF_VAR_*, STELLAR_RPC_URL
docker compose up -d          # postgres, redis, localstack(SQS/secrets), web, api, 4 engines (full parity)
# cloud:
terraform init -backend-config=backend.hcl
terraform plan -var-file=envs/staging/terraform.tfvars
# kubernetes:
kubectl config use-context <eks-context>
kubectl apply -k kubernetes/namespaces/    # staging + prod quotas + default-deny
```

Full deploy, CI/CD and contract deploy: `docs/DEPLOYMENT_GUIDE.md`.

---

## Code Quality Gates (all repos)

```bash
# app
pnpm lint && pnpm typecheck && pnpm test && pnpm build

# contract
npm run lint && npm run typecheck && npm run test && npm run build

# infra
terraform fmt -check -recursive && terraform validate
docker compose config -q
```

CI enforces the same gates (`.github/workflows/` per repo; `stellariq-infra` holds the reusable workflows consumed by the other two).

Pre-commit: Husky + `lint-staged` runs Prettier + ESLint on staged files. Do not `--no-verify`.

---

## SDK Example

```ts
import { createClient } from "@stellariq/sdk"
const cli = createClient({ baseUrl: "http://localhost:4000", apiKey: process.env.STELLARIQ_KEY })
const { assets } = await cli.assets.list({ search: "XLM" })
const price = await cli.prices.get("XLM")           // { price, confidence, timestamp, sources }
const quote = await cli.quotes.get({ from: "XLM", to: "USDC", amount: "10000" })
const ws = cli.ws()
ws.subscribe("XLM/USDC:price", (e) => console.log(e.price))
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `ECONNREFUSED 5432` in contract | `docker compose up -d postgres && docker compose ps` (healthy?) + check `DATABASE_URL` |
| `READY 503` from app `/ready` | `contract` internal-api `:4110` not up — start `stellariq-contract` first (app degrades to free tier + mock) |
| `x-ratelimit-*` `429` locally | Redis not running or `REDIS_URL` wrong — `docker compose up -d redis` |
| `STELLAR_RPC_URL` errors | Check `.env` testnet URL; rate-limited fallback is automatic retry with backoff — see `apps/indexer/src/rpc.ts` |
| `terraform init` fails | Missing `backend.hcl` (copy from `backend.hcl.example`) or `aws sso login` required |

## Where to read more

* `docs/ARCHITECTURE.md` (topology), `docs/DATA_PIPELINE.md` (indexer -> routing), `docs/API_SPEC.md` (REST+WS), `docs/DEPLOYMENT_GUIDE.md` (how to ship), `PRD.md:34` (roadmap).
