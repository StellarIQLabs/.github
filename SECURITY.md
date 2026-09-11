# Security Policy

## Supported Versions

| Project | Versions | Supported |
|---------|----------|-----------|
| `stellariq-app`, `stellariq-contract`, `stellariq-infra`, `.github` docs | `main` | Yes — only `main` receives fixes until first tagged release |
| Tagged releases `v*` (post-MVP) | latest `v*` + previous minor | Yes |

## Reporting a Vulnerability

**Do NOT open a public issue.**

Use one of these private channels:

1.  **GitHub Private Reporting** — `Security` tab -> `Report a vulnerability` on any `StellarIQLabs/*` repo (preferred, gives you a private fork + advisory draft).
2.  Email the maintainers via the contact listed on the org profile.

Include:

* Repo and commit / tag
* Steps to reproduce (PoC if possible)
* Impact assessment (data integrity, key leakage, fund risk, availability)
* Suggested fix if you have one

### What happens next

```
  Report received
      |
      v
  Triage (<48h) -> confirm severity
      |
      v
  Private fix branch -> review -> tests
      |
      v
  Coordinated release + advisory + CVE if applicable
      |
      v
  Reporter credited (if desired)
```

* We will acknowledge within **48 hours**.
* We will keep you updated on triage and timeline.
* We will credit you in the advisory unless you prefer to remain anonymous.
* We ask for **90-day coordinated disclosure**; we will ship the fix and publish the advisory on the agreed date.

## Scope & Threat Model

StellarIQ is **non-custodial** `PRD.md:129` — it never holds private keys. The security boundary is:

```
  Wallet (private keys)  --->  StellarIQ API (unsigned XDR, quotes, analytics)
        | signer                     | no key material, only builders
        v                            v
     Stellar Network <--- signed Tx submitted by user/wallet directly
```

What we treat as high severity:

* Private key exfiltration or signing bypass
* Unsigned-transaction tampering that swaps route/amount undetected
* Auth bypass, rate-limit bypass, secret leakage from `terraform/` or `kubernetes/secrets/`
* Injection via `DexAdapter` or indexer that corrupts prices/liquidity
* Remote code execution in any Docker image

See `docs/SECURITY_MODEL.md` for the full model, dependency scanning, and secret-rotation policy.

## Security Hardening In-Repo

* `dependabot` + `npm audit` / `cargo audit` + image scanning in CI (`.github/workflows/security-scan.yml` in `stellariq-infra`)
* `terraform` secrets via AWS Secrets Manager, synced to K8s by `ExternalSecrets` — no secrets in git (`kubernetes/secrets/external-secrets.yaml`)
* Branch protection on `main`, signed commits encouraged
* Pre-commit: `gitleaks` style secret scan via Husky where configured

## Past Advisories

None yet — first advisories will appear as GitHub Security Advisories linked from each repo.
