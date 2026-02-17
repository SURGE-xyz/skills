# surge-openclaw

Full AI agent skill for the SURGE token launchpad.

## Capabilities

| Feature | Endpoints |
|---------|-----------|
| Wallet Management | Create EVM/Solana wallets, one-time free funding, balance checks |
| Token Launch | Deploy on Base (EVM) or Solana with bonding curve |
| Trading | Buy/sell with auto-routing (pre-DEX + post-DEX) |
| Transfers | Native coins (ETH/SOL) and tokens (ERC-20/SPL) |
| Ownership | Transfer or renounce ERC-20 contract ownership |
| Monitoring | Token balances, tx status, trade history |

## Supported Chains

- **Base** (EVM) — Aerodrome DEX
- **Solana** — Raydium CPMM

## Authentication

Requires an API key (`X-API-Key: sk-surge-...`) from [app.surge.xyz](https://app.surge.xyz).

## Usage

See [`SKILL.md`](./SKILL.md) for the complete agent instruction set.
