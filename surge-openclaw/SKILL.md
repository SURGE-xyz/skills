---
name: SURGE OpenClaw
description: "Create, trade, and transfer tokens on SURGE via API. Full lifecycle — wallet setup, one-time free funding, token launch, trading (pre-DEX + post-DEX), transfers (native + tokens), ownership management, portfolio monitoring, and trade history. Base URL: back.surge.xyz."
author: SURGE
version: 8.0.0
tags: [token-launch, trading, transfers, ownership, defi, evm, solana, base, raydium, launchpad, bonding-curve, aerodrome, post-dex]
auth:
  type: api-key
  header: X-API-Key
  management: "User creates keys at app.surge.xyz → Profile → API Keys"
---

# SURGE OpenClaw — AI Agent Skill

API base URL: **https://back.surge.xyz**.

You are helping the user create and trade tokens on the SURGE launchpad. This document tells you **everything** you need to know. Follow it step by step.

---

## TL;DR — What This Skill Does

- User can **create a token** on EVM (Base) or Solana — for **free** (one time)
- After creation, user can **buy and sell** tokens — both pre-DEX (bonding curve / launchpad) **and** post-DEX (Aerodrome on Base / Raydium CPMM on Solana)
- **Trading auto-routes** by token phase — same endpoints work for both pre-DEX and post-DEX
- User can **transfer** native coins (ETH/SOL) and tokens (ERC-20/SPL) to any address
- User can **transfer or renounce ownership** of ERC-20 token contracts
- All wallets are **server-managed** — no private keys needed from user
- **Token balance**, **transaction status**, and **trade history** endpoints let you monitor everything
- You (the AI agent) handle the entire process through API calls

---

## Quick Reference — All Endpoints

| Method | Path | What it does |
|--------|------|-------------|
| `GET` | `/openclaw/launch-info` | Live config: fees, chains, categories |
| `POST` | `/openclaw/wallet/create` | Create/get EVM wallet |
| `POST` | `/openclaw/wallet/create-solana` | Create/get Solana wallet |
| `GET` | `/openclaw/wallet/:walletId` | Wallet info (DB) |
| `GET` | `/openclaw/wallet/:walletId/balance` | On-chain native balance |
| `POST` | `/openclaw/wallet/:walletId/token-balance` | ERC-20 / SPL token balance |
| `POST` | `/openclaw/wallet/:walletId/fund` | One-time free funding |
| `GET` | `/openclaw/wallet/:walletId/history` | Paginated trade history |
| `POST` | `/openclaw/tx-status` | Check transaction status |
| `POST` | `/openclaw/launch` | Launch EVM token |
| `POST` | `/openclaw/launch-solana` | Launch Solana token |
| `POST` | `/openclaw/token-status` | Check EVM token phase |
| `POST` | `/openclaw/quote` | Price quote (post-DEX only) |
| `POST` | `/openclaw/buy` | Buy EVM tokens |
| `POST` | `/openclaw/sell` | Sell EVM tokens |
| `POST` | `/openclaw/buy-solana` | Buy Solana tokens |
| `POST` | `/openclaw/sell-solana` | Sell Solana tokens |
| `POST` | `/openclaw/transfer/native-evm` | Transfer native EVM coin (ETH/BNB) |
| `POST` | `/openclaw/transfer/erc20` | Transfer ERC-20 tokens |
| `POST` | `/openclaw/transfer/solana` | Transfer SOL or SPL tokens |
| `POST` | `/openclaw/erc20/transfer-ownership` | Transfer ERC-20 contract ownership |
| `POST` | `/openclaw/erc20/renounce-ownership` | Renounce ERC-20 contract ownership (irreversible) |

Every request requires the header: `X-API-Key: sk-surge-...`

---

## Before You Start — Checklist

Before doing anything, make sure you have:

| # | What | How to check |
|---|------|-------------|
| 1 | **API Key** | User gives you a key starting with `sk-surge-...`. If they don't have one — see "How to Get an API Key" below. |
| 2 | **Working API** | Call `GET /openclaw/launch-info`. If 401 — key is bad. If 200 — you're good. |

### How to Get an API Key

If the user doesn't have an API key yet, tell them exactly this:

> **To get your API key:**
> 1. Go to [app.surge.xyz](https://app.surge.xyz)
> 2. Sign up / log in (you can use your wallet, email, or social login)
> 3. Go to **Profile → API Keys**
> 4. Click **Generate** and give it a name (e.g. "My Agent")
> 5. **Copy the key immediately** — it's shown only once!
> 6. Give the key to me and I'll handle everything from here
>
> You can have up to 5 active keys. If you lose one, revoke it and create a new one.

Once you have the key, proceed to Step 0.

---

## Step 0: Load Configuration (do this silently)

Call this **before talking to the user about tokens**:

```
GET /openclaw/launch-info
Header: X-API-Key: {key}
Base URL: https://back.surge.xyz
```

This gives you live data — fees, chains, categories. **Never hardcode these values.**

You now know:
- Which chains are available and their fees
- `minBalance` — how much the wallet needs
- Available categories for tokens
- File size limits

If this fails with **401** — tell the user their API key is invalid and point them to app.surge.xyz → Profile → API Keys.

Response:
```json
{
  "chains": [
    {
      "chainId": "1",
      "chainName": "Base",
      "networkId": "8453",
      "chainType": "EVM",
      "fee": "0.005",
      "feeRaw": "5000000000000000",
      "feeSymbol": "ETH",
      "estimatedGas": "0.00003",
      "minBalance": "0.00536"
    },
    {
      "chainId": "3",
      "chainName": "Solana",
      "networkId": "solana",
      "chainType": "SOLANA",
      "fee": "0.1",
      "feeRaw": "100000000",
      "feeSymbol": "SOL",
      "minBalance": "0.136",
      "defaults": {
        "supply": 1000000000,
        "decimals": 6,
        "totalSellPercent": 80,
        "fundraisingGoal": { "SOL": "85", "USD1": "12500" },
        "fundraisingMints": ["SOL", "USD1"]
      }
    }
  ],
  "categories": ["ai", "infrastructure", "meme", "rwa", "defi", "privacy", "derivatives", "gamefi", "robotics", "depin", "desci", "healthcare", "education", "socialfi"],
  "limits": {
    "maxImageSize": "5MB",
    "maxDocSize": "100MB",
    "allowedImageTypes": ["image/png", "image/jpeg", "image/jpg", "image/webp"],
    "allowedDocTypes": ["application/pdf"]
  }
}
```

---

## Step 1: Create a Wallet

The user needs a server-managed wallet. One wallet per chain type (EVM or Solana).

**For EVM (Base):**
```
POST /openclaw/wallet/create
Header: X-API-Key: {key}
```

**For Solana:**
```
POST /openclaw/wallet/create-solana
Header: X-API-Key: {key}
```

Response:
```json
{
  "walletId": "vun3srwayi6z1h8momm83tdr",
  "address": "0xD29c...Be2E",
  "chainType": "EVM",
  "needsFunding": true,
  "isNew": true
}
```

- Save `walletId` — you'll need it for everything
- If `isNew: false` — wallet already existed, that's fine
- The wallet is linked to the user's account

**Tell the user:**
> I've created a server-managed wallet for you. No private keys to worry about — everything is handled securely on our side.

---

## Step 2: Fund the Wallet (ONE TIME FREE)

**Important rule: The platform funds each wallet exactly ONCE for free.** This covers the gas + minimum buy for the first token launch. After that, the user pays for everything themselves.

```
POST /openclaw/wallet/{walletId}/fund
Header: X-API-Key: {key}
```

**If successful** (`needsFunding: false`):
> Your wallet has been funded! You get one free token launch from SURGE — gas and fees are covered.

**If already funded** (second call):
> This wallet was already funded. The free launch is a one-time gift. If you need more funds (for trading, etc.), send ETH/SOL directly to your wallet address: `{address}`

**If funding fails** (e.g. platform funder is low):
> Automatic funding didn't go through right now. You can either:
> 1. Try again in a few minutes
> 2. Send funds manually to your wallet address: `{address}` on the Base network
>
> The minimum balance needed is **{minBalance}** (from launch-info).

Success response:
```json
{
  "walletId": "...",
  "needsFunding": false,
  "funding": [{ "chain": "Base", "amount": "0.006", "txHash": "0x...", "success": true }]
}
```

Already funded response:
```json
{
  "needsFunding": false,
  "funding": [],
  "message": "Wallet already funded. For additional funds, send directly to the wallet address."
}
```

---

## Step 3: Check Balance

```
GET /openclaw/wallet/{walletId}/balance
Header: X-API-Key: {key}
```

Response includes `sufficient: true/false` and `minRequired` per chain.

**If `sufficient: true`** — proceed to token creation.

**If `sufficient: false`** — tell the user:
> Your wallet balance is **{balance}** but you need at least **{minRequired}**. Please send **{minRequired - balance}** to your wallet address: `{address}` on **{chain}**.

---

## Step 4: Collect Token Information

Now collect what's needed to launch the token. **Ask one group at a time. Don't dump everything at once.**

**Fixed parameters (NOT configurable):**
- **Total supply** — always **1,000,000,000** (1 billion). Cannot be changed.
- **Decimals** — fixed per chain (18 for EVM, 6 for Solana). Cannot be changed.

Do NOT ask the user about supply or decimals. These are set by the platform.

### 4a. Name, Ticker, Description (REQUIRED)

Ask:
> Let's create your token! I need three things to start:
> 1. **Token name** — the full name (e.g. "NeuralNet Token")
> 2. **Ticker** — short symbol, 3-5 letters (e.g. "NNET")
> 3. **Description** — one sentence about what the token is for (e.g. "Decentralized AI compute marketplace")

Rules:
- If user gives only a ticker — ask for the full name
- If user gives a paragraph as description — say: "Great, I'll use that as the detailed description. Can you give me a one-liner tagline too?"
- **Never make up names, tickers, or descriptions yourself**

### 4b. Logo Image (REQUIRED)

Ask:
> Now I need a logo for your token. Send me a **direct link** to an image file (PNG, JPG, or WEBP, max 5MB).

**Validate before accepting:**
- URL must start with `http://` or `https://`
- Must be a **direct file link**, not a gallery page
  - GOOD: `https://i.imgur.com/abc123.png`
  - BAD: `https://imgur.com/abc123` (gallery page)
  - BAD: `https://drive.google.com/file/d/...` (preview page, not direct)

### 4c. Chain (REQUIRED)

Build this question from launch-info data:

> Which blockchain do you want to launch on?
>
> Available:
> - **Base** (EVM) — fee: {fee} ETH
> - **Solana** — fee: {fee} SOL
>
> Base is recommended for most projects. Solana is great for high-speed, low-cost trading.

Map the user's choice to the correct `chainId` from launch-info. The user should never see raw IDs.

**This determines everything else:**
- EVM → use `/launch`, needs `ethAmount`
- Solana → use `/launch-solana`, needs `preBuyAmount`

### 4d. Initial Buy Amount (REQUIRED)

**If EVM chain:**

Ask:
> How much ETH do you want for the initial buy? This buys the very first tokens when your contract launches.
>
> - Minimum: slightly more than **{fee} ETH** (the contract fee)
> - Recommended: **{fee x 2} ETH** for a good start
> - This determines the starting price — bigger amount = higher starting price

| User says | What to do |
|-----------|-----------|
| Less than or equal to fee | "The contract fee is {fee} ETH. Your amount must be higher. I'd recommend {fee x 2}." |
| Reasonable (0.01-1 ETH) | Accept |
| Very large (10+ ETH) | "That's {amount} ETH — are you sure?" |

**If Solana chain:**

First ask which currency:
> Which currency do you want for your token's fundraising pool?
>
> - **SOL** (default) — fundraising goal: 85 SOL. Your initial buy is in SOL.
> - **USD1** (stablecoin) — fundraising goal: 12,500 USD1. Your initial buy is in USD1.
>
> Most projects use SOL. USD1 is for stablecoin-based pools.

Then ask for the amount:
> How much {SOL/USD1} for the initial buy? Even a small amount like **0.01** works.

**Important about funding:**
- If user picks **SOL** — the one-time free funding covers platform fee + gas + min preBuy. They're ready to go.
- If user picks **USD1** — the free funding only covers SOL (for platform fee + gas). The user **must have USD1 tokens in their wallet** for the initial buy. Tell them:
  > Since you chose USD1, you'll need USD1 tokens in your Solana wallet for the initial buy. The platform covers SOL fees, but USD1 you provide yourself. Send USD1 to your wallet address: `{address}`

### 4e. Category (OPTIONAL — but suggest it)

Based on what the user already told you, suggest 2-3 matching categories:

> Based on your description, this sounds like an **AI** project. Should I set the category to `ai`? Or is it more `defi` / `infrastructure`?

Available categories: `ai`, `infrastructure`, `meme`, `rwa`, `defi`, `privacy`, `derivatives`, `gamefi`, `robotics`, `depin`, `desci`, `healthcare`, `education`, `socialfi`

**Don't list all 14.** Just suggest the best 2-3 matches.

### 4f. Optional Extras

Ask once:
> Want to add any extras? All optional — you can add them later on app.surge.xyz:
> - **Banner image** (wide header, 1200x400 ideal)
> - **Detailed description** (markdown supported)
> - **Pitch deck or whitepaper** (PDF, up to 100MB)
> - **Social links** (website, Twitter/X, Telegram, Discord, GitHub)
> - **Team description**

If user says "no" / "skip" — move to confirmation.

**For files (banner, pitch deck, whitepaper):** ask the user for a direct link to the file.

**Convert social handles automatically:**
- `@neuralnet` → `https://x.com/neuralnet`
- `t.me/neuralnet` → `https://t.me/neuralnet`
- `discord.gg/abc` → `https://discord.gg/abc`

---

## Step 5: Confirm Before Launch

**ALWAYS show a summary. ALWAYS wait for explicit "yes".**

> Here's your token ready to launch:
>
> **Token:** {name} ({ticker})
> **Description:** {description}
> **Category:** {category or "not set"}
> **Chain:** {chainName}
> **Initial buy:** {ethAmount} ETH (chain fee: {fee} ETH)
> **Logo:** {logoUrl}
> **Wallet:** {walletId}
> {any other filled fields}
>
> **This is irreversible.** Once launched, the token goes live immediately.
>
> Ready to launch? (yes/no)

- Only proceed after explicit "yes" / "go" / "launch" / "do it"
- If user says "change X" — update that field and show summary again

---

## Step 6: Launch

**EVM:**
```
POST /openclaw/launch
Header: X-API-Key: {key}
Content-Type: application/json

{
  "name": "NeuralNet Token",
  "ticker": "NNET",
  "description": "Decentralized AI compute marketplace",
  "logoUrl": "https://i.imgur.com/abc123.png",
  "chainId": "CHAIN_ID_FROM_LAUNCH_INFO",
  "walletId": "YOUR_WALLET_ID",
  "ethAmount": "0.01"
}
```

**Solana:**
```
POST /openclaw/launch-solana
Header: X-API-Key: {key}
Content-Type: application/json

{
  "name": "NeuralNet Token",
  "ticker": "NNET",
  "description": "Decentralized AI compute marketplace",
  "logoUrl": "https://i.imgur.com/abc123.png",
  "chainId": "SOLANA_CHAIN_ID_FROM_LAUNCH_INFO",
  "walletId": "YOUR_SOLANA_WALLET_ID",
  "preBuyAmount": "0.5"
}
```

**Optional fields** (both EVM and Solana): `bannerUrl`, `fullDescription`, `projectName`, `category`, `pitchDeckUrl`, `whitepaperUrl`, `websiteLink`, `githubLink`, `telegramLink`, `discordLink`, `xLink`, `teamShortDescription`, `teamMembers`

**Solana-only optional:** `fundraisingMint` (`"SOL"` or `"USD1"`, default `"SOL"`)

---

## Step 7: After Launch

**EVM success response:**
```json
{
  "txHash": "0x...",
  "metadataUid": "...",
  "chainId": "1",
  "caip2": "eip155:8453",
  "explorerUrl": "https://basescan.org/tx/0x...",
  "tokenName": "NeuralNet Token",
  "tokenTicker": "NNET",
  "chainName": "Base",
  "summary": "Token NeuralNet Token (NNET) launched on Base. Initial buy: 0.01 ETH. Tx: https://basescan.org/tx/0x..."
}
```

**Tell the user** — use the `summary` field, plus:
> **View your token on SURGE:** https://app.surge.xyz/trade/{tokenAddress}
> Transaction: {explorerUrl}
> It may take a minute to appear on the platform. Your token is now trading on the bonding curve — anyone can buy and sell!

**Important:** The SURGE platform link (`app.surge.xyz/trade/{tokenAddress}`) must always come first — before the block explorer link.

**Solana success response:**
```json
{
  "signature": "5xK9...",
  "mint": "7pB8z...",
  "metadataUid": "...",
  "chainId": "3",
  "explorerUrl": "https://solscan.io/tx/5xK9...",
  "tokenName": "NeuralNet Token",
  "tokenTicker": "NNET",
  "chainName": "Solana",
  "summary": "Token NeuralNet Token (NNET) launched on Solana. Tx: https://solscan.io/tx/5xK9..."
}
```

**Tell the user** — use `summary`, plus:
> **View your token on SURGE:** https://app.surge.xyz/trade/{mint}
> Transaction: {explorerUrl}

**Important:** The SURGE platform link (`app.surge.xyz/trade/{mint}`) must always come first — before the block explorer link.

---

## Trading (After Token Launch)

After a token is launched, you can buy and sell. **Trading requires the wallet to have funds — the free funding only covers the first launch.**

Tell the user:
> To trade, you'll need funds in your wallet. Send ETH/SOL to your wallet address: `{address}`

### How Trading Works — Auto-Routing

**You don't need to choose the trading method.** The API automatically detects the token phase and routes your trade:

- **Pre-DEX (bonding curve / launchpad)** — Trade directly on the SURGE bonding curve (EVM) or Raydium launchpad (Solana)
- **Post-DEX (Aerodrome / Raydium CPMM)** — After graduation (100% bonding curve), the token migrates to a DEX. The API swaps automatically via Aerodrome (Base) or Raydium CPMM (Solana)

**Same endpoints, same parameters, regardless of phase.** Just call `/buy` or `/sell`.

### EVM Trading — Check Token Phase (optional)

```
POST /openclaw/token-status
{ "chainId": "1", "tokenAddress": "0x..." }
```

| Phase | Meaning | How `/buy` and `/sell` work |
|-------|---------|-----------|
| `bonding_curve` | Pre-DEX, active trading | Trades via BondingETHRouter (bonding curve) |
| `migrated_to_dex` | Graduated to Aerodrome DEX | Trades via Aerodrome Router (DEX swap with `agentToken`) |
| `not_launched` | Not active | Cannot trade — returns error |

The response also includes `tokenName`, `tokenTicker`, `price`, `marketCap`, `liquidity`, `agentTokenAddress`. You can use `tokenName`/`tokenTicker` to display the token to the user. You don't need `agentTokenAddress` — the API handles routing automatically.

### EVM Price Quote (Post-DEX only)

```
POST /openclaw/quote
{
  "chainId": "1",
  "tokenAddress": "0x...",
  "amount": "0.01",
  "side": "buy"
}
```

Returns `amountIn`, `amountOut`, `phase`. Only works for `migrated_to_dex` tokens. `side` must be `"buy"` or `"sell"`.

### Buy EVM Tokens

```
POST /openclaw/buy
{
  "chainId": "1",
  "walletId": "abc123",
  "tokenAddress": "0x...",
  "ethAmount": "0.01",
  "amountOutMin": "0"
}
```

Works for **both** bonding curve and post-DEX. The API detects the phase automatically.

**Success response (201):**
```json
{
  "txHash": "0x...",
  "explorerUrl": "https://basescan.org/tx/0x...",
  "chainId": "1",
  "chainName": "Base",
  "action": "buy",
  "phase": "bonding_curve",
  "tokenAddress": "0x...",
  "tokenName": "Dev Test Token",
  "tokenTicker": "DTT",
  "amountIn": "0.01",
  "amountInUnit": "ETH",
  "priceAfter": "0.000000001234",
  "marketCapAfter": "1.42",
  "liquidityAfter": "2.84",
  "walletId": "abc123",
  "summary": "Bought DTT for 0.01 ETH on Base via bonding curve (pre-DEX). Price: ... Tx: https://basescan.org/tx/0x..."
}
```

**How to present this to the user:**
Use the `summary` field as the main message. Optionally show `explorerUrl` as a link, and `tokenName`/`tokenTicker`/`priceAfter`/`marketCapAfter` for extra detail.

### Sell EVM Tokens

```
POST /openclaw/sell
{
  "chainId": "1",
  "walletId": "abc123",
  "tokenAddress": "0x...",
  "tokenAmount": "1000",
  "amountOutMin": "0"
}
```

**Success response (201):**
```json
{
  "txHash": "0x...",
  "explorerUrl": "https://basescan.org/tx/0x...",
  "chainId": "1",
  "chainName": "Base",
  "action": "sell",
  "phase": "bonding_curve",
  "tokenAddress": "0x...",
  "tokenName": "Dev Test Token",
  "tokenTicker": "DTT",
  "amountIn": "1000",
  "amountInUnit": "DTT",
  "priceAfter": "0.000000000933",
  "marketCapAfter": "1.07",
  "liquidityAfter": "2.14",
  "walletId": "abc123",
  "summary": "Sold 1000 DTT on Base via bonding curve (pre-DEX). Price: ... Tx: https://basescan.org/tx/0x..."
}
```

Notes:
- Auto-approves ERC-20 allowance if needed (waits for on-chain confirmation before selling)
- Don't sell too much at once — can cause revert on small pools
- `amountOutMin: "0"` = no slippage protection (use `/quote` to set a reasonable value for post-DEX)
- For post-DEX sells: you're selling the `agentToken`, but pass the original `tokenAddress` — the API resolves it

### Buy Solana Tokens

```
POST /openclaw/buy-solana
{
  "chainId": "3",
  "walletId": "sol_wallet_id",
  "mintAddress": "7pB8z...",
  "solAmount": "0.01"
}
```

Works for **both** Raydium launchpad (pre-DEX) and CPMM (post-DEX).

**Success response (201):**
```json
{
  "signature": "5xK9...",
  "chainId": "3",
  "action": "buy",
  "mintAddress": "7pB8z...",
  "amount": "0.01",
  "amountUnit": "SOL",
  "explorerUrl": "https://solscan.io/tx/5xK9...",
  "chainName": "Solana",
  "summary": "Bought tokens for 0.01 SOL on Solana (launchpad). Tx: https://solscan.io/tx/5xK9..."
}
```

### Sell Solana Tokens

```
POST /openclaw/sell-solana
{
  "chainId": "3",
  "walletId": "sol_wallet_id",
  "mintAddress": "7pB8z...",
  "tokenAmount": "1000"
}
```

**Success response (201):**
```json
{
  "signature": "3mN7...",
  "chainId": "3",
  "action": "sell",
  "mintAddress": "7pB8z...",
  "amount": "1000",
  "amountUnit": "tokens",
  "explorerUrl": "https://solscan.io/tx/3mN7...",
  "chainName": "Solana",
  "summary": "Sold 1000 tokens on Solana (launchpad). Tx: https://solscan.io/tx/3mN7..."
}
```

Notes:
- Gas fees paid by platform (gas sponsorship)
- 5% slippage applied automatically
- `mintAddress` = the `mint` from launch response
- Post-DEX: uses Raydium Transaction API for CPMM swap

---

## Transfers

After a token launch or trade, the user may want to **send tokens or native coins to another address**. Transfers move funds **from the server-managed wallet to any external address**.

**Important:** Transfers require the wallet to have sufficient balance for the amount + gas fees. The API checks balances before sending and returns a clear error if insufficient.

### Transfer Native EVM Coin (ETH/BNB)

Send native currency (ETH on Base, BNB on BSC) to another address.

```
POST /openclaw/transfer/native-evm
Header: X-API-Key: {key}
Content-Type: application/json

{
  "chainId": "1",
  "walletId": "abc123",
  "toAddress": "0x742d35Cc6634C0532925a3b844Bc9e7595f2bD18",
  "nativeAmount": "0.01"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `chainId` | string | Database chain ID (from `/launch-info`) |
| `walletId` | string | Privy EVM wallet ID |
| `toAddress` | string | Recipient Ethereum address (0x...) |
| `nativeAmount` | string | Amount in ETH/BNB (human-readable, e.g. `"0.01"`) |

**Success response:**
```json
{
  "txHash": "0x...",
  "explorerUrl": "https://basescan.org/tx/0x...",
  "walletId": "abc123",
  "toAddress": "0x742d...bD18",
  "amount": "0.01",
  "amountUnit": "native"
}
```

**Balance check:** The API verifies `nativeBalance >= amount + estimated gas` before sending. If insufficient:
> "Your wallet doesn't have enough ETH. You need approximately {required} wei (amount + gas), but only have {available} wei. Send more ETH to `{address}` on Base."

### Transfer ERC-20 Tokens

Send ERC-20 tokens to another address. The API auto-fetches token decimals.

```
POST /openclaw/transfer/erc20
Header: X-API-Key: {key}
Content-Type: application/json

{
  "chainId": "1",
  "walletId": "abc123",
  "tokenAddress": "0xTokenContractAddress...",
  "toAddress": "0x742d35Cc6634C0532925a3b844Bc9e7595f2bD18",
  "tokenAmount": "1000"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `chainId` | string | Database chain ID |
| `walletId` | string | Privy EVM wallet ID |
| `tokenAddress` | string | ERC-20 token contract address (0x...) |
| `toAddress` | string | Recipient Ethereum address (0x...) |
| `tokenAmount` | string | Amount in human-readable format (e.g. `"1000"`, NOT raw/wei) |

**Success response:**
```json
{
  "txHash": "0x...",
  "explorerUrl": "https://basescan.org/tx/0x...",
  "walletId": "abc123",
  "tokenAddress": "0xToken...",
  "toAddress": "0x742d...bD18",
  "amount": "1000",
  "amountUnit": "tokens"
}
```

**Balance checks (two):**
1. Token balance — must have enough tokens to transfer
2. Native balance — must have enough ETH/BNB for gas

### Transfer SOL or SPL Tokens (Solana)

Send SOL or SPL tokens on Solana. If `mintAddress` is provided — transfers SPL tokens. Otherwise — transfers SOL.

**Transfer SOL:**
```
POST /openclaw/transfer/solana
Header: X-API-Key: {key}
Content-Type: application/json

{
  "chainId": "3",
  "walletId": "sol_wallet_id",
  "toAddress": "7hmbbq...aV9o",
  "amount": "0.5"
}
```

**Transfer SPL tokens:**
```
POST /openclaw/transfer/solana
Header: X-API-Key: {key}
Content-Type: application/json

{
  "chainId": "3",
  "walletId": "sol_wallet_id",
  "toAddress": "7hmbbq...aV9o",
  "amount": "1000",
  "mintAddress": "7pB8z..."
}
```

| Field | Type | Description |
|-------|------|-------------|
| `chainId` | string | Database chain ID for Solana |
| `walletId` | string | Privy Solana wallet ID |
| `toAddress` | string | Recipient address (Base58, 32-44 chars) |
| `amount` | string | Amount to transfer (human-readable) |
| `mintAddress` | string (optional) | SPL token mint address. If provided → SPL transfer. If omitted → SOL transfer |

**SOL transfer response:**
```json
{
  "signature": "5xK9...",
  "explorerUrl": "https://solscan.io/tx/5xK9...",
  "walletId": "sol_wallet_id",
  "toAddress": "7hmbbq...aV9o",
  "amount": "0.5",
  "amountUnit": "SOL"
}
```

**SPL transfer response:**
```json
{
  "signature": "3mN7...",
  "explorerUrl": "https://solscan.io/tx/3mN7...",
  "walletId": "sol_wallet_id",
  "toAddress": "7hmbbq...aV9o",
  "amount": "1000",
  "amountUnit": "tokens",
  "tokenMint": "7pB8z..."
}
```

**Balance checks:**
- SOL transfer: `SOL balance >= amount + tx fee`
- SPL transfer: `token balance >= amount` AND `SOL balance >= tx fee + ATA rent` (if recipient doesn't have an associated token account, one is auto-created — costs ~0.002 SOL rent)

**Tell the user:**
> Transfer sent! {amount} {SOL/tokens} sent to `{toAddress}`.
> View transaction: {explorerUrl}

---

## Ownership Management (ERC-20)

These endpoints let the user **transfer or renounce ownership** of an ERC-20 token contract. Only works on tokens that implement the OpenZeppelin `Ownable` interface (all SURGE-launched tokens do).

### Transfer Ownership

Transfer the `owner` role of an ERC-20 contract to a new address. The new owner can then call `onlyOwner` functions.

```
POST /openclaw/erc20/transfer-ownership
Header: X-API-Key: {key}
Content-Type: application/json

{
  "chainId": "1",
  "walletId": "abc123",
  "tokenAddress": "0xTokenContractAddress...",
  "newOwner": "0xNewOwnerAddress..."
}
```

| Field | Type | Description |
|-------|------|-------------|
| `chainId` | string | Database chain ID |
| `walletId` | string | Privy EVM wallet ID (must be current owner) |
| `tokenAddress` | string | ERC-20 token contract address |
| `newOwner` | string | New owner Ethereum address (0x...) |

**Success response:**
```json
{
  "txHash": "0x...",
  "explorerUrl": "https://basescan.org/tx/0x...",
  "walletId": "abc123",
  "tokenAddress": "0xToken...",
  "newOwner": "0xNewOwner...",
  "method": "transferOwnership"
}
```

**Tell the user:**
> Ownership of `{tokenAddress}` has been transferred to `{newOwner}`.
> This is reversible — the new owner can transfer it back.
> Tx: {explorerUrl}

### Renounce Ownership ⚠️ IRREVERSIBLE

Permanently remove the owner from an ERC-20 contract. After this, **no one** can call `onlyOwner` functions ever again. The `owner()` becomes `0x0000...0000`.

**ALWAYS warn the user before calling this. ALWAYS wait for explicit confirmation.**

```
POST /openclaw/erc20/renounce-ownership
Header: X-API-Key: {key}
Content-Type: application/json

{
  "chainId": "1",
  "walletId": "abc123",
  "tokenAddress": "0xTokenContractAddress..."
}
```

| Field | Type | Description |
|-------|------|-------------|
| `chainId` | string | Database chain ID |
| `walletId` | string | Privy EVM wallet ID (must be current owner) |
| `tokenAddress` | string | ERC-20 token contract address |

**Success response:**
```json
{
  "txHash": "0x...",
  "explorerUrl": "https://basescan.org/tx/0x...",
  "walletId": "abc123",
  "tokenAddress": "0xToken...",
  "method": "renounceOwnership"
}
```

**Before calling — ALWAYS show this warning and wait for "yes":**
> ⚠️ **This is irreversible.** Renouncing ownership means:
> - No one will ever be able to call owner-only functions on this token
> - This cannot be undone — there is no way to reclaim ownership
> - This is a trust signal to the community ("the contract is immutable")
>
> Token: `{tokenAddress}` on {chainName}
>
> Are you absolutely sure? (yes/no)

**After success:**
> Ownership of `{tokenAddress}` has been renounced. The contract is now ownerless — no one can modify it.
> Tx: {explorerUrl}

### Ownership Errors

| Error message | What to tell the user | Agent action |
|--------------|----------------------|-------------|
| `"You're not owner of this token"` | "Your wallet is not the current owner of this token. Only the owner can transfer or renounce ownership." | Check if user has the right wallet/token |
| `"Failed to verify ownership: contract may not support Ownable interface"` | "This token contract doesn't support ownership management. Only tokens with the Ownable interface (all SURGE-launched tokens) support this." | No action — incompatible contract |
| `"transferOwnership failed: ..."` / `"renounceOwnership failed: ..."` | Use the error message. Likely insufficient gas or RPC issue. | Check balance, retry once |

---

## Amount Formats — CRITICAL

The agent **must** follow these format rules. Wrong format causes validation errors or on-chain reverts.

| Field | Format | Example | WRONG |
|-------|--------|---------|-------|
| `tokenAmount` (EVM sell, Solana sell) | Human-readable number string | `"1000"`, `"100000"`, `"500.5"` | `"100000000000000000000000"` (wei) |
| `ethAmount` (EVM buy/launch) | ETH amount as string | `"0.01"`, `"0.005"` | wei or raw |
| `solAmount` (Solana buy) | SOL amount as string | `"0.5"`, `"0.01"` | lamports |
| `preBuyAmount` (Solana launch) | SOL/USD1 amount as string | `"0.5"`, `"100"` | lamports |
| `amount` (quote) | Amount as string | `"0.01"`, `"1000"` | — |
| `nativeAmount` (EVM native transfer) | ETH/BNB amount as string | `"0.01"`, `"0.1"` | wei or raw |
| `amount` (Solana transfer) | SOL or token amount as string | `"0.5"`, `"1000"` | lamports |

**Validation rules (server rejects if violated):**
- Pattern: `^(0|[1-9]\d*)(\.\d+)?$` — positive numbers only, no leading zeros (except `"0.x"`), no negative, no letters
- Max length: 30 characters
- Good: `"0.01"`, `"1000"`, `"100000.5"`, `"0"`
- Bad: `"-1"`, `"00.5"`, `"1,000"`, `"1e18"`, `"abc"`, `"100000000000000000000000"` (too long)

**Address formats:**
- EVM `tokenAddress`: valid Ethereum address (0x + 40 hex chars, checksum)
- Solana `mintAddress`: Base58 string, 32-44 chars (charset: `1-9A-HJ-NP-Za-km-z`, no `0OIl`)

The backend converts human amounts to contract units internally. Passing raw/wei causes "arithmetic underflow or overflow" or a wrong-sized trade.

---

## Portfolio & Monitoring

### Check Token Balance

Check how many tokens the wallet holds for a specific token.

```
POST /openclaw/wallet/{walletId}/token-balance
{
  "chainId": "1",
  "tokenAddress": "0x..."
}
```

**Response:**
```json
{
  "walletId": "abc123",
  "address": "0xD29c...Be2E",
  "chainType": "EVM",
  "tokenAddress": "0x...",
  "tokenName": "Dev Test Token",
  "tokenSymbol": "DTT",
  "balance": "1500.0",
  "balanceRaw": "1500000000000000000000",
  "decimals": 18
}
```

| Field | Description |
|-------|-------------|
| `balance` | Human-readable balance (e.g. `"1500.0"`) — show this to the user |
| `balanceRaw` | Raw on-chain value in smallest units — don't show to user |
| `tokenName` | Token name from contract (EVM) or mint address (Solana) |
| `tokenSymbol` | Token symbol from contract (EVM) or empty string (Solana) |
| `decimals` | Token decimal places |

**Solana note:** If the wallet has no associated token account (never held this token), returns `balance: "0"`, `balanceRaw: "0"`, `decimals: 0`.

**When to use this:**
- Before selling: check if the user actually holds enough tokens
- After buying: verify tokens arrived
- Portfolio display: show the user what they hold

### Check Transaction Status

Check if a transaction is confirmed, pending, or failed.

```
POST /openclaw/tx-status
{
  "chainId": "1",
  "txHash": "0x..."
}
```

**Response:**
```json
{
  "txHash": "0x...",
  "chainId": "1",
  "status": "confirmed",
  "blockNumber": 42095060,
  "gasUsed": "234567",
  "message": "Transaction confirmed on-chain."
}
```

| Status | Meaning | What to do |
|--------|---------|-----------|
| `pending` | Not yet mined / not found | Wait and poll again in 5-10 seconds |
| `confirming` | Solana only: in block but not finalized | Wait and poll again in 3-5 seconds |
| `confirmed` | Success — mined and finalized | Done. Show explorer link to user |
| `failed` | Reverted or errored | Explain failure, suggest retry with different params |

**EVM-specific fields:** `blockNumber`, `gasUsed`
**Solana-specific fields:** `slot`, `confirmationStatus` (e.g. `"finalized"`, `"confirmed"`)

**When to use this:**
- After any trade/launch if the result seems uncertain
- When the user asks "did my transaction go through?"
- For debugging: if the response had a `txHash` / `signature` but something seems off

### Trade History

Get paginated list of all trades and launches for a wallet.

```
GET /openclaw/wallet/{walletId}/history?limit=20&offset=0
```

| Param | Default | Max | Description |
|-------|---------|-----|-------------|
| `limit` | 20 | 100 | Number of records |
| `offset` | 0 | — | Skip first N records |

**Response:**
```json
{
  "trades": [
    {
      "id": "42",
      "userId": "123",
      "walletId": "abc123",
      "chainId": "1",
      "chainName": "Base",
      "chainType": "EVM",
      "action": "buy",
      "tokenAddress": "0x...",
      "tokenName": "Dev Test Token",
      "tokenTicker": "DTT",
      "amount": "0.01",
      "amountUnit": "ETH",
      "txHash": "0x...",
      "explorerUrl": "https://basescan.org/tx/0x...",
      "success": true,
      "errorCode": null,
      "createdAt": "2026-02-13T10:44:27.000Z",
      "updatedAt": "2026-02-13T10:44:27.000Z"
    }
  ],
  "total": 15
}
```

`action` values: `"buy"`, `"sell"`, `"launch"`, `"transfer"`

**When to use this:**
- User asks "what did I do?" or "show me my trades"
- Verifying past operations
- Building a portfolio summary

---

## Error Handling — Complete Reference

### HTTP Status Codes

| Status | Meaning | Agent action |
|--------|---------|-------------|
| **200/201** | Success | Parse response, show to user |
| **400** | Bad Request — validation error or known failure | Read `code` and `message` from body. Use error code table below |
| **401** | Unauthorized — bad API key | Tell user: "Your API key is invalid or expired. Go to app.surge.xyz → Profile → API Keys to generate a new one." |
| **403** | Forbidden — user blocked/banned | **DO NOT RETRY.** Read `message` for ban details. See "Blocked User" section below |
| **429** | Too Many Requests — rate limited | Back off. Wait 10-30 seconds and try once. If still 429, wait longer. **Do NOT loop rapidly** — repeated 429s lead to auto-ban |
| **500** | Server error | "Server error — not your fault. Let's try again in a minute." Retry once after 30s |

### 403 — Blocked User

When a user is auto-banned for rate limit abuse, the response looks like:

```json
{
  "statusCode": 403,
  "message": "Account suspended until 2026-02-14T12:00:00.000Z. Reason: rate_limit_abuse. Contact support.",
  "error": "Forbidden"
}
```

**Ban types:**
- **Temp ban (1h or 24h):** Message contains `"until <ISO date>"`. Tell the user when they can try again. DO NOT retry before that time.
- **Permanent ban:** Message contains `"permanently"`. All API keys are deactivated. Tell the user to contact support.

**What to tell the user:**
> Your account has been temporarily suspended {until / permanently} due to rate limit violations. {If temp: "You can try again after {time}."} {If permanent: "Please contact SURGE support to resolve this."}

**Agent must NEVER:**
- Retry requests after a 403
- Try to work around a ban (different endpoint, etc.)
- Hide the ban from the user

### 400 — Error Codes (All Operations)

When any operation fails with a known error, the API returns 400 with a structured body:

```json
{ "statusCode": 400, "message": "Human-readable error description", "code": "ERROR_CODE" }
```

**Master error code table** — covers trades, launches, EVM, and Solana:

| Code | What happened | What to tell the user | Agent action |
|------|--------------|----------------------|-------------|
| `INSUFFICIENT_BALANCE` | Wallet doesn't have enough ETH/SOL/tokens for the operation + gas | "Your wallet doesn't have enough {ETH/SOL/tokens}. Try a smaller amount or send funds to your wallet: `{address}`" | Call `/wallet/{id}/balance` to show current balance |
| `CONTRACT_REVERT` | Smart contract rejected the transaction (EVM) | "The transaction was rejected — possible causes: slippage too high, or the pool has low liquidity. Try a smaller amount." | Retry with smaller amount or higher `amountOutMin` |
| `GAS_ESTIMATION_FAILED` | Can't estimate gas — usually means tx would fail (EVM) | "Transaction would fail (not enough balance for amount + gas). Use a smaller amount or add funds." | Check balance, suggest top-up |
| `APPROVAL_FAILED` | ERC-20 token approval failed (EVM sell only) | "Token approval failed. Your wallet may need more ETH for gas. Top up and retry." | Check ETH balance |
| `NONCE_ERROR` | A transaction is already pending for this wallet (EVM) | "A transaction is already in progress. Wait 15-30 seconds and try again." | Wait 15s, retry once |
| `SIMULATION_FAILED` | Transaction simulation failed (Solana) | "Transaction simulation failed — not enough balance, too much slippage, or low pool liquidity. Try a smaller amount." | Retry with smaller amount |
| `TX_EXPIRED` | Blockhash expired before confirmation (Solana) | "Transaction expired. This is normal on Solana when the network is busy. Let me try again." | Retry immediately (auto-gets new blockhash) |
| `NO_ROUTE` | No trading pool found for this token (Solana) | "No trading route found. The token may not have a liquidity pool yet." | Check `/token-status` to verify the token exists |
| `CONTRACT_ERROR` | On-chain program rejected the tx (Solana) | "The on-chain program rejected the transaction. Try a smaller amount." | Retry with smaller amount |
| `TRADE_FAILED` | Generic trade failure (fallback) | Use the `message` from response as-is, or: "Transaction failed. Try a smaller amount or check your balance." | Check balance, retry with smaller amount |
| `LAUNCH_FAILED` | Generic launch failure (fallback) | Use the `message` from response as-is, or: "Token launch failed. Check wallet balance and try again." | Check balance |

### Common 400 Errors (non-code)

These come from validation or business logic, not from on-chain errors:

| Error message contains | What to say | Agent action |
|------------------------|-------------|-------------|
| `"image"` or `"download"` | "I couldn't download your logo. Make sure it's a **direct link** to an image file (PNG, JPG, WEBP), not a gallery page." | Ask for new URL |
| `"explicit"` | "Your image was flagged as inappropriate. Please use a different image." | Ask for new image |
| `"fee"` or `"ethAmount"` | "The amount is too low. The fee is **{fee}** — your amount must be higher. Try **{fee x 2}**." | Suggest correct amount |
| `"not a Solana wallet"` | "You're using an EVM wallet for Solana. Let me create a Solana wallet." | Call `/wallet/create-solana` |
| `"Wallet not found"` | "That wallet doesn't exist. Let me create a new one." | Call `/wallet/create` or `/wallet/create-solana` |
| `"already funded"` | "Your wallet was already funded with the free launch. For more funds, send to: `{address}`" | No action needed |
| `"insufficient funds"` | "Your wallet doesn't have enough balance. Send at least **{minBalance}** to `{address}` on **{chain}**." | Show balance, suggest top-up |
| `"arithmetic underflow"` | The amount format is wrong. `tokenAmount` must be human-readable (e.g. `"1000"`), NOT wei. Or the sell amount is larger than the pool can handle. | Fix format and retry, or use smaller amount |
| `"Daily funding limit"` | "Platform funding is temporarily unavailable. Send funds manually to `{address}`." | Give wallet address |
| `"Insufficient native balance"` | "Not enough ETH/SOL to cover the transfer amount plus gas fees. Send more to `{address}`." | Show balance, suggest top-up |
| `"Insufficient token balance"` | "You don't have enough tokens to transfer. Check your balance first." | Call `/wallet/{id}/token-balance` |
| `"Insufficient SOL for fees"` | "Not enough SOL to cover transaction fees (and ATA rent if needed). Top up with SOL." | Show SOL balance |
| `"You're not owner"` | "Your wallet is not the owner of this token contract." | Check correct wallet/token |
| `"not support Ownable interface"` | "This token doesn't support ownership management." | No retry — incompatible contract |
| `"Native transfer failed"` | "Transfer failed. Check that the recipient address is valid and you have enough balance." | Check balance, retry once |
| `"ERC-20 transfer failed"` | "Token transfer failed. Check balance and gas." | Check both balances |
| `"transferOwnership failed"` | "Ownership transfer failed. Ensure you have enough ETH for gas." | Check gas balance |
| `"renounceOwnership failed"` | "Renounce failed. Ensure you have enough ETH for gas." | Check gas balance |

---

## Recovery Playbooks

### Trade failed with INSUFFICIENT_BALANCE

```
1. Call GET /openclaw/wallet/{walletId}/balance
2. Show user: "Your balance is {balance}. You need at least {amount + gas estimate}."
3. If EVM: "Send ETH to {address} on Base network"
   If Solana: "Send SOL to {address}"
4. Wait for user to confirm they've sent funds
5. Re-check balance
6. Retry the trade
```

### Trade failed with NONCE_ERROR

```
1. Tell user: "A transaction is already processing. Let me wait a moment."
2. Wait 15 seconds
3. Retry the exact same trade once
4. If still NONCE_ERROR: "There's a stuck transaction. Wait 1-2 minutes and try again."
```

### Solana trade failed with TX_EXPIRED

```
1. Tell user: "The transaction expired — this happens when Solana is busy. Retrying now."
2. Retry immediately (the API generates a fresh blockhash automatically)
3. If it fails again: wait 10 seconds, retry once more
4. If third failure: "Solana network is congested. Try again in a few minutes."
```

### Transaction seems stuck (pending for too long)

```
1. Call POST /openclaw/tx-status with the txHash/signature
2. If "pending": wait 10 seconds, poll again (up to 5 times)
3. If "confirmed": show success to user
4. If "failed": explain what went wrong
5. If still "pending" after 5 polls: "Transaction is taking longer than usual. 
   You can check its status here: {explorerUrl}"
```

### User asks "how many tokens do I have?"

```
1. Call POST /openclaw/wallet/{walletId}/token-balance
   Body: { "chainId": "...", "tokenAddress": "..." }
2. Show: "You hold {balance} {tokenSymbol} ({tokenName})"
3. If balance is "0": "You don't hold any of this token yet."
```

### User asks "what trades did I make?"

```
1. Call GET /openclaw/wallet/{walletId}/history?limit=10
2. Format as a list:
   - "Bought DTT for 0.01 ETH — Feb 13, 2026 10:44 AM — confirmed"
   - "Sold 1000 DTT — Feb 13, 2026 10:50 AM — confirmed"
3. If total > limit: "Showing last {limit} of {total} trades."
```

### Rate limited (429)

```
1. DO NOT immediately retry
2. Wait 10 seconds
3. Retry once
4. If still 429: wait 30 seconds, retry once
5. If still 429: "We're being rate limited. Let's wait a minute before trying again."
6. WARNING: Do NOT loop rapidly — 10 consecutive 429s in 5 minutes triggers auto-ban
```

### User is blocked (403)

```
1. Read the message to determine ban type
2. If message contains "until <date>":
   - Parse the date
   - Tell user: "Your account is temporarily suspended until {date} due to {reason}. 
     You can try again after that time."
3. If message contains "permanently":
   - Tell user: "Your account has been permanently suspended. 
     Please contact SURGE support to resolve this."
4. STOP all API calls. DO NOT retry. DO NOT try other endpoints.
```

### Transfer failed — insufficient balance

```
1. Identify which balance is insufficient:
   - Native transfer: need enough ETH/SOL for amount + gas
   - ERC-20 transfer: need enough tokens AND enough ETH for gas
   - Solana SPL: need enough tokens AND enough SOL for fee + possible ATA rent
2. Call GET /openclaw/wallet/{walletId}/balance to show native balance
3. If token transfer: call POST /openclaw/wallet/{walletId}/token-balance to show token balance
4. Tell user exactly what's missing:
   - "You have {balance} ETH but need ~{required} for amount + gas. Send {shortfall} ETH to {address}"
   - "You have {balance} tokens but need {amount}. You don't have enough to transfer."
5. Wait for user to top up, re-check, retry
```

### Ownership operation failed

```
1. If "You're not owner": 
   - The wallet is not the current owner of this contract
   - Ask user: "Are you sure this is the right token address? Your wallet {address} is not the owner."
2. If "not support Ownable interface":
   - The contract doesn't implement Ownable — nothing to do
   - Tell user: "This token doesn't support ownership management."
3. If "transferOwnership failed" / "renounceOwnership failed":
   - Usually a gas issue — check native balance
   - Call GET /openclaw/wallet/{walletId}/balance
   - If balance is low: "You need more ETH for gas. Send some to {address}."
   - If balance is fine: retry once
```

### User asks "send my tokens to..."

```
1. Determine what they want to send:
   - Native coin (ETH/SOL) → use /transfer/native-evm or /transfer/solana
   - Specific token → use /transfer/erc20 or /transfer/solana with mintAddress
2. Identify the chain from their wallet (EVM vs Solana)
3. Get the recipient address from the user
4. Get the amount from the user
5. Check balance before sending:
   - For native: GET /wallet/{walletId}/balance
   - For tokens: POST /wallet/{walletId}/token-balance
6. Confirm with user before sending:
   > Sending {amount} {token/ETH/SOL} to {toAddress} on {chain}.
   > Gas fees will be deducted from your wallet.
   > Proceed? (yes/no)
7. Send the transfer
8. Show explorerUrl to user
```

### User asks to renounce ownership

```
1. ALWAYS warn first — this is irreversible:
   > ⚠️ Renouncing ownership is PERMANENT. No one will ever be able to call 
   > owner-only functions on this contract again. Are you absolutely sure?
2. Wait for explicit "yes" / "do it" / "renounce"
3. Call POST /openclaw/erc20/renounce-ownership
4. Show result with explorerUrl
5. Confirm: "Ownership renounced. The contract is now ownerless."
```

### Launch failed — what to check

```
1. Check the error code:
   - INSUFFICIENT_BALANCE → check balance, suggest top-up
   - CONTRACT_REVERT → check that ethAmount > fee
   - GAS_ESTIMATION_FAILED → wallet needs more ETH
   - NONCE_ERROR → wait 15s, retry
   - SIMULATION_FAILED (Solana) → check SOL balance
   - TX_EXPIRED (Solana) → retry immediately
2. If error code is LAUNCH_FAILED (generic):
   - Check balance first
   - If balance is fine, retry once
   - If retry fails: "Launch failed. This might be a temporary issue. Try again in a few minutes."
```

---

## API Key Management

These endpoints use **JWT auth** (not API key) — available at app.surge.xyz:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/openclaw/api-keys` | `POST` | Generate new key. Body: `{ "name": "My Bot" }`. Max 5 active, 20 lifetime. |
| `/openclaw/api-keys` | `GET` | List all keys (shows prefix only, never the full key) |
| `/openclaw/api-keys/:keyId` | `DELETE` | Revoke a key |

The agent typically doesn't call these — the user manages keys at app.surge.xyz.

---

## Rate Limits

Every endpoint has per-user rate limits. Exceeding them returns 429. **Repeated 429s lead to progressive auto-ban.**

| Endpoint | Limit (per minute) |
|----------|--------------------|
| `GET /launch-info` | 30 |
| `POST /launch`, `/launch-solana` | 5 |
| `POST /wallet/create`, `/wallet/create-solana` | 5 |
| `GET /wallet/:id` | 20 |
| `GET /wallet/:id/balance` | 10 |
| `POST /wallet/:id/token-balance` | 20 |
| `POST /wallet/:id/fund` | 3 |
| `POST /tx-status` | 20 |
| `POST /token-status`, `/quote` | 20 |
| `POST /buy`, `/sell`, `/buy-solana`, `/sell-solana` | 10 |
| `POST /transfer/native-evm`, `/transfer/erc20`, `/transfer/solana` | 10 |
| `POST /erc20/transfer-ownership`, `/erc20/renounce-ownership` | 10 |
| `GET /wallet/:id/history` | 20 |

**Progressive auto-ban system:**
- 10 rate limit violations within 5 minutes triggers auto-ban
- Strike 1: blocked for **1 hour** (auto-unblocks)
- Strike 2: blocked for **24 hours** (auto-unblocks)
- Strike 3+: **permanent** block (all API keys deactivated, admin unblock only)
- Strike count persists — it does not reset after a temp ban expires

**Agent rule:** Always respect rate limits. If you get a 429, wait before retrying. Never loop rapidly on any endpoint.

---

## Agent Rules (Non-Negotiable)

| # | Rule |
|---|------|
| 1 | **Never invent data.** Don't make up names, tickers, descriptions, or URLs. Always ask. |
| 2 | **Never hardcode fees or chain IDs.** Always use `GET /launch-info`. |
| 3 | **Always confirm before launch.** Show summary, wait for "yes". Irreversible. |
| 4 | **Suggest, don't list.** For categories — suggest 2-3, don't dump 14. |
| 5 | **Validate URLs.** Direct image link, not gallery page. |
| 6 | **Convert handles.** `@name` → `https://x.com/name` |
| 7 | **2-3 questions per message.** Don't overwhelm. |
| 8 | **Don't re-ask.** If user mentioned info earlier, use it. |
| 9 | **Never ask for private keys.** Wallets are server-managed. |
| 10 | **One free launch.** After that, user funds manually. Be clear about this upfront. |
| 11 | **Translate errors.** Never show raw JSON errors. Use the error tables above. |
| 12 | **Help with file uploads.** Ask the user for a direct link to the image or file. |
| 13 | **Amount format.** `tokenAmount` = human number string. Never wei/raw/lamports. |
| 14 | **Respect rate limits.** Never loop rapidly. Back off on 429. Repeated violations = auto-ban. |
| 15 | **Never retry on 403.** A 403 means the user is banned. Explain and stop. |
| 16 | **Use summary field.** Trade and launch responses include a `summary` string — use it as the main message to the user. |
| 17 | **Verify with token-balance.** Before selling, check the user's token balance. Before buying, check native balance. |
| 18 | **Poll tx-status for stuck transactions.** If a trade response has a hash but something seems off, check status. |
| 19 | **Show explorer links.** Every trade and launch response has `explorerUrl` — always share it with the user. |
| 20 | **Trade history for context.** If the user asks about past trades or what happened, use `/wallet/{id}/history`. |
| 21 | **Confirm transfers.** Before sending any transfer, show the user: amount, recipient, chain. Wait for "yes". |
| 22 | **Double-confirm renounce.** Renouncing ownership is **irreversible**. Always explain consequences and wait for explicit confirmation. Never auto-renounce. |
| 23 | **Check balance before transfers.** Before any transfer, verify the wallet has enough funds (native + gas for ERC-20, token + SOL for SPL). |
| 24 | **Right wallet for right chain.** EVM transfers need EVM wallet. Solana transfers need Solana wallet. If mismatch — create the correct wallet first. |
