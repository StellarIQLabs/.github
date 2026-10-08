<p align="center">
  <img src="assets/logo.svg" width="96" alt="StellarIQ Give logo" />
</p>

# StellarIQ Labs

### Transparent charity donations on Stellar

**StellarIQ Give** lets anyone donate to a charity campaign from their own
wallet. The money goes straight to the charity's wallet in the same
transaction, and every donation leaves a public receipt on the Stellar network.
No middleman holds the funds, and anyone can check where they went.

### Live demo (testnet)

- **Web app:** **WEB_URL**
- **API docs (Swagger):** https://stellariq-api-p1hz.onrender.com/docs
- **Campaigns API:** https://stellariq-api-p1hz.onrender.com/v1/campaigns
- **Donations contract:** [`CBCHKIDR...F2KX`](https://stellar.expert/explorer/testnet/contract/CBCHKIDRFJ4KO2DGJEP75NJPYN65YVD6QOVVHC5IU7PRTHGHW75OF2KX)
- **Swap router contract:** [`CC277AA6...VHSP`](https://stellar.expert/explorer/testnet/contract/CC277AA6E6WZIQRA4N45TQ3O6VV5MUSDMRZCNHO43QENMYXV6E5OVHSP)

### How it works

```
  Donor wallet (Freighter)
        |  signs one transaction
        v
  donations contract  --->  token moves donor -> charity wallet
        |                    receipt stored + donation_made event
        v
  StellarIQ API  --->  web app shows progress, donors and receipts
        ^
  stellariq-data indexes events and prices; router converts other assets
```

### Repos

| Repo | Role |
|------|------|
| [`stellariq-app`](https://github.com/StellarIQLabs/stellariq-app) | Web app, API, SDK and design system |
| [`stellariq-contract`](https://github.com/StellarIQLabs/stellariq-contract) | Soroban contracts: donations and the swap router |
| [`stellariq-data`](https://github.com/StellarIQLabs/stellariq-data) | Indexer, price feeds and analytics used to value and convert donations |
| [`stellariq-infra`](https://github.com/StellarIQLabs/stellariq-infra) | Terraform, Docker, Kubernetes, monitoring and deploy scripts |
| [`.github`](https://github.com/StellarIQLabs/.github) | Org health files and shared docs |

### Contributing

Issues labelled `good first issue` and `help wanted` are open across the repos.
Read [`CONTRIBUTING.md`](https://github.com/StellarIQLabs/.github/blob/main/CONTRIBUTING.md)
to get started.
