# Deployment Guide

## Topology

```
  ECR (per-service images)  <-  .github/workflows/build-images.yml  (reused by app + contract)
        |
        v
  EKS 1.30 (terraform/modules/cluster, autoscaler)  <-  RDS 16 + Proxy  <-  ElastiCache Redis  <-  SQS (modules/queues)
        |
        +-- namespaces: stellariq-staging (quotas) , stellariq-prod (quotas + default-deny net policies)
        +-- jobs/db-migrate (pre-rollout) , demo-seed
        +-- deploy/web, deploy/api (HPA, env from ExternalSecrets), indexer (stateful, PVC), price-engine, analytics-engine (+ CronJobs), routing-engine (latency-tuned + HPA)
        +-- ingress (TLS cert-manager/Let's Encrypt, rate-limit + DDoS guards)
        |
     ALB -> api.stellariq.xyz + app.stellariq.xyz
        |
     monitoring (Prometheus scrapes all) -> Grafana SLO -> Loki logs -> alerts -> uptime probes -> consistency reconciler
```

Local parity is identical via `stellariq-infra/docker-compose.yml` (postgres, redis, localstack for SQS/secrets, web, api, 4 engines).

Performance SLOs `PRD.md:1121`: API cached `<300ms`, quote `<1s`, dashboard `<2s`; availability `99.9%` `PRD.md:1130`. Headroom without redesign `PRD.md:1148`: `10+ protocols, 100k assets, 1M swaps/day, 10k API users`.

---

## Local Parity (docker compose)

```bash
cd stellariq-infra
cp .env.example .env        # never commit .env
docker compose up -d        # postgres, redis, localstack, web, api, indexer, price-engine, analytics-engine, routing-engine
docker compose ps           # all healthy?
curl http://localhost:4000/health   # app
curl http://localhost:4110/health   # contract internal-api
curl http://localhost:4000/v1/markets
./scripts/smoke.sh          # end-to-end (hits /v1/markets, /v1/quote, ws)
```

---

## Cloud — Terraform

```bash
cd stellariq-infra/terraform
cp backend.hcl.example backend.hcl        # edit: bucket, key, region, dynamodb_table for state locking
cp soroban.tfvars.example soroban.tfvars  # STELLAR_RPC_URL, passphrase per env
terraform init -backend-config=backend.hcl
terraform fmt -check -recursive
terraform validate
terraform plan  -var-file=envs/staging/terraform.tfvars
terraform apply -var-file=envs/staging/terraform.tfvars
# production uses envs/production/terraform.tfvars with larger sizing
```

Modules wiring (`main.tf`): `networking` -> `postgres` (RDS 16 + Proxy + Secrets Manager), `redis` (ElastiCache), `queues` (SQS + DLQs), `backups` (AWS Backup daily + KMS + alarm), `secrets`, `registry` (ECR per service + S3 WASM bucket), `cluster` (EKS 1.30 + node group + autoscaler). Remote state is S3 + DynamoDB (`providers.tf`).

---

## Kubernetes

```bash
aws sso login --profile <profile> && aws eks update-kubeconfig --name stellariq-<env> --region <region>
kubectl apply -k kubernetes/namespaces/          # staging + prod quotas + default-deny
kubectl apply -f kubernetes/secrets/external-secrets.yaml   # Secrets Manager -> K8s (no secrets in git)
kubectl apply -f kubernetes/jobs/db-migrate.yaml  # drizzle migrations before workloads
kubectl apply -f kubernetes/web/ kubernetes/api/ kubernetes/indexer/ \
                 kubernetes/price-engine/ kubernetes/analytics-engine/ \
                 kubernetes/routing-engine/ kubernetes/ingress/
kubectl rollout status deployment/stellariq-api -n stellariq-prod
kubectl get hpa -n stellariq-prod
```

Images are ECR per service (`registry` module), tagged by commit SHA from CI.

### Rollout helpers

```bash
./scripts/bootstrap.sh           # first cluster bring-up (namespaces, secrets, migrations, workloads, ingress)
./scripts/rollout.sh <service>   # kubectl set image + rollout status + smoke
./scripts/rollback.sh <service>  # kubectl rollout undo
./scripts/smoke.sh               # curl /v1/markets, /v1/quote, ws subscribe, simulator
```

---

## CI/CD

Reusable workflows live in `stellariq-infra/.github/workflows/` and are called by `stellariq-app` and `stellariq-contract`:

* `build-images.yml` — multi-arch, layer-cached builds, push to ECR with SHA tag
* `test.yml` — lint + typecheck + test + build + `terraform fmt/validate`
* `security-scan.yml` — `npm audit` / `cargo audit` + dependency + image scans
* `deploy-staging.yml` — auto on `push` to `main` when tests + scans pass
* `deploy-production.yml` — manual approval gate (`environment: production`), then `db-migrate` job -> rollout -> `smoke.sh` -> auto-rollback on probe failure
* `shared-docs.yml` — docs lint for `.github`

Staging is continuous; production is guarded.

---

## Contract Deployment

```bash
cd stellariq-infra
./scripts/soroban-networks.sh testnet     # sets STELLAR_RPC_URL + passphrase
./scripts/deploy-contracts.sh             # wraps stellar-cli, deploys contracts/Cargo.toml workspace
./scripts/soroban-networks.sh mainnet && ./scripts/deploy-contracts.sh --network mainnet
```

The simulator (`services/simulator/`) is deployed alongside the API and called pre-submission: `PRD.md:1144`.

---

## Monitoring & SLOs

```
  Prometheus (scrapes /metrics from every service)
    -> Grafana dashboards (SLO): API p95, quote latency, ingestion lag, TVL drift
    -> Loki (via promtail) -> logs
    -> alerts (alertmanager -> Slack/PagerDuty)
    -> uptime probes (synthetic GET /health + /v1/prices/XLM every 30s)
    -> consistency reconciler (pool reserves vs on-chain, metric tvl_drift)
    -> tracing (OpenTelemetry + Sentry)  see monitoring/tracing/
  perf/ k6: api-p95.js, dashboard.js, swaps.js  (gates: p95 <300ms, quote <1s, dashboard <2s)
```

Key alerts: `rpc_throttled`, `ingestion_lag_seconds`, `price_confidence <0.9`, `no-recent-backup` (from `backups` module), `api_availability <99.9%`.

---

## Disaster Recovery

See `stellariq-infra/docs/disaster-recovery.md` (RPO/RTO, backup verification, cluster rebuild) and `docs/promotion.md` (staging -> prod promotion checklist) and `docs/launch-readiness.md` (MVP launch gates `PRD.md:1391`).

Quick DR check:

```bash
aws backup list-recovery-points --backup-vault-name stellariq
kubectl get cronjob -n stellariq-prod  # monthly secret rotation + daily backups
```

---

## Checklist Before `terraform apply` to Production

* [ ] `terraform fmt` + `validate` + `plan` reviewed
* [ ] `CHANGELOG.md` bumped in app + contract
* [ ] `smoke.sh` green in staging
* [ ] `BACKUP` and `no-recent-backup` alarms green in Grafana
* [ ] `ExternalSecrets` synced (`kubectl get externalsecrets -A`)
