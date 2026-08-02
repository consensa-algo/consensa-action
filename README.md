# Consensa Clearance (GitHub Action)

Consensa Clearance is a GitHub Action that pays a micro-fee to clear your repository's open-source dependencies through Consensa on Algorand, producing an on-chain consent + attribution receipt on every release.
Every payment settles atomically into a four-way split — provider 80%, upstream maintainers' escrow 11% (claimable, per-dependency attribution on-chain), commons 7%, protocol 2% — capped by a per-repository spending limit checked against the payer wallet's own on-chain history *before* every payment, not trusted from local or server-side state.
The settlement contract's conservation invariant (payment == provider + upstream + commons + protocol) is enforced on-chain at every transaction and covered by property-based fuzz testing — proven, not merely asserted.

## Prerequisites

- An Algorand **Mainnet** account funded with a small amount of ALGO (network fees) and USDC (payment + your chosen spending cap). This wallet should be dedicated to this repository's own clearance payments — never reuse a Consensa-operated wallet.
- That account's mnemonic added as a **repository secret** named `CONSENSA_PAYER_MNEMONIC` (Settings → Secrets and variables → Actions). GitHub encrypts repository secrets at rest; this Action never logs or persists the mnemonic.
- A dependency manifest at `manifest_path` (default `package.json`). v1 supports `package.json` only.

## Usage

Add `CONSENSA_PAYER_MNEMONIC` as a repository secret, then add this workflow:

```yaml
name: Consensa clearance

on:
  release:
    types: [published]
  workflow_dispatch: {}

jobs:
  clearance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Consensa clearance
        uses: consensa-algo/consensa-action@v1
        with:
          endpoint: https://consensa-endpoint-production.up.railway.app
          manifest_path: package.json
          spend_cap_usd: "1.00"  # raising this is an explicit decision
        env:
          CONSENSA_PAYER_MNEMONIC: ${{ secrets.CONSENSA_PAYER_MNEMONIC }}
```

### Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `endpoint` | yes | — | Consensa clearance endpoint **host only**, e.g. `https://consensa-endpoint-production.up.railway.app` — do **not** include `/v1/clearance`; the Action appends it itself |
| `manifest_path` | no | `package.json` | Path to the dependency manifest (v1 supports `package.json`) |
| `spend_cap_usd` | no | `1.00` | Max USD this repository may spend per rolling 24h, enforced on-chain before paying |

### Outputs

| Name | Description |
|---|---|
| `payment_tx_id` | Confirmed payment transaction id |
| `settle_tx_id` | Settlement/allocation transaction id (empty on the atomic path) |
| `receipt_url` | Reconciliation URL for this receipt |
| `status` | `settled` \| `allocation-pending` \| `pending` |
