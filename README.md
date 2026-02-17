<h1 align="center">SURGE Skills</h1>

<p align="center">
  <strong>AI Agent skills for the SURGE token launchpad</strong>
</p>

<p align="center">
  <a href="https://surge.xyz">Website</a> &middot;
  <a href="https://app.surge.xyz">App</a> &middot;
  <a href="https://x.com/Surgexyz_">X (Twitter)</a>
</p>

---

## What is SURGE?

SURGE is a token launchpad that lets you create, trade, and manage tokens on **Base** (EVM) and **Solana** — with bonding curve mechanics, automatic DEX migration, and server-managed wallets.

## What are Skills?

Skills are structured instructions that teach AI agents how to interact with external services. Each skill is a Markdown file (`SKILL.md`) containing everything an agent needs: API endpoints, request/response examples, error handling, and step-by-step workflows.

Any AI agent that supports skill-based tooling (OpenClaw, Bankr, custom agents) can read these files and autonomously execute the full SURGE workflow — from wallet creation to token launches and trading.

## Available Skills

| Skill | Description | Chains |
|-------|-------------|--------|
| **[surge-openclaw](./surge-openclaw/)** | Full SURGE API skill — wallet setup, token launch, trading (pre-DEX + post-DEX), transfers, ownership management, portfolio & history | Base, Solana |

## Quick Start

### 1. Get an API Key

1. Go to [app.surge.xyz](https://app.surge.xyz)
2. Sign up / log in
3. Navigate to **Profile → API Keys**
4. Click **Generate** — copy the key immediately (shown only once)

### 2. Install the Skill

**For AI agents with skill support** (OpenClaw, Bankr, Codex, or any agent that supports GitHub-based skills): tell your agent to install the skill from `https://github.com/SURGE-xyz/skills`.

**For any other agent or custom setup:** copy the contents of [`surge-openclaw/SKILL.md`](./surge-openclaw/SKILL.md) into your agent's skill/prompt context.

### 3. Start Using

Tell your agent:

- *"Create a token called NeuralNet (NNET) on Base"*
- *"Buy 0.01 ETH of NNET"*
- *"Send 500 NNET to 0x..."*
- *"Show my trade history"*

The skill handles everything: wallet creation, funding, trading, transfers, and error recovery.

## What the Skill Covers

- **Wallet Management** — create & fund server-managed wallets (EVM + Solana)
- **Token Launch** — deploy tokens on Base or Solana with bonding curve
- **Trading** — buy/sell with automatic routing (pre-DEX bonding curve → post-DEX Aerodrome/Raydium)
- **Transfers** — send native coins (ETH/SOL) and tokens (ERC-20/SPL) to any address
- **Ownership** — transfer or renounce ERC-20 contract ownership
- **Monitoring** — token balances, transaction status, trade history
- **Error Handling** — complete error code reference with recovery playbooks

## Repository Structure

```
skills/
├── README.md                 # This file
├── LICENSE
└── surge-openclaw/
    └── SKILL.md              # Full AI agent skill (production)
```

## Authentication

All API requests require an API key passed in the `X-API-Key` header:

```
X-API-Key: sk-surge-...
```

Keys are managed at [app.surge.xyz → Profile → API Keys](https://app.surge.xyz). Each user can have up to 5 active keys.

## Contributing

We welcome contributions! If you'd like to improve an existing skill or add a new one:

1. Fork this repository
2. Create your branch (`git checkout -b improve-skill`)
3. Make your changes
4. Submit a Pull Request

Please follow the existing skill format and ensure all examples are accurate.

## License

This project is licensed under the MIT License — see the [LICENSE](./LICENSE) file for details.

---

<p align="center">
  Built by <a href="https://surge.xyz">SURGE</a>
</p>
