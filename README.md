# Ethy AI — Agent Skills

Composable skills that teach AI coding agents (Claude Code, Cursor, Cline, Codex, Amp, Antigravity, and 8 more) how to use [Ethy AI](https://ethyai.app)'s on-demand market intelligence, autonomous trading agents, and automation engine.

All paid endpoints settle via the **OKX Agent Payments Protocol** on X Layer — gas-free, sub-second confirmations, no signup, no API key. Pay per call from any OKX Agentic Wallet.

## Install

```bash
npx skills add EthyAI/ethy-skills
```

This installs the skills under `.agents/skills/` in your current directory and symlinks them for every supported agent.

## Skills in this bundle

| Skill | Status | What it does |
|---|---|---|
| [`ethy-paid-data`](./ethy-paid-data/SKILL.md) | ✅ Live | Fetch BUY/SELL signals, technical indicators, Ethy Score, and LLM chart analysis. Pay per call ($0.05–$0.50). |
| `ethy-trading` | Roadmap | Execute spot trades on Base / X Layer via Ethy's agent (with stops + automations). |
| `ethy-automations` | Roadmap | Create, pause, and modify rule-based automations through natural language. |
| `ethy-backtest` | Roadmap | Backtest a strategy spec against historical OHLCV data. |
| `ethy-perpetuals` | Roadmap | Open / close / manage perp positions on Hyperliquid through Ethy. |

## Prerequisites

Each skill declares its own prerequisites in its `SKILL.md`. For the live `ethy-paid-data` skill you need:

1. **OKX OnchainOS CLI** — install via OKX's own skill bundle:
   ```bash
   npx skills add okx/onchainos-skills
   ```
   Then log into your OKX Agentic Wallet:
   ```bash
   onchainos wallet login your.email@example.com
   onchainos wallet verify <code-from-email>
   ```

2. **USD₮0 on X Layer** — fund your wallet address (visible via `onchainos wallet addresses`) with USD₮0 (contract `0x779ded0c9e1022225f8e0630b35a9b54be713736`). $5 covers ~100 cheap calls.

3. **A supported agent** — Claude Code, Cursor, Cline, Codex, Amp, or any of the other 8 universal targets. The `skills` installer auto-detects which agents you have and symlinks accordingly.

## Quick start

After installing both skill bundles (`EthyAI/ethy-skills` + `okx/onchainos-skills`) and logging in, open your agent and ask a market question:

```
> Should I long ETH right now? Use Ethy's paid intelligence if needed.
```

The agent will:

1. Detect the `ethy-paid-data` skill is relevant.
2. Climb the spend ladder cheapest-first — typically start with `signal` ($0.05) or `score` ($0.10).
3. Prompt you to confirm each charge before fetching.
4. Synthesize the results and report what it thinks, with a running cost total.

A typical "should I trade X?" question costs **$0.10–$0.70 USD₮0** end-to-end. The OKX Agent Payments Protocol skill handles all the signing automatically — you only see the confirmation prompts.

## How payments work

Ethy's paid endpoints respond with `HTTP 402` on the first call. Your agent's payment skill (`okx-agent-payments-protocol`, installed automatically when you install `okx/onchainos-skills`) detects the 402, asks you to confirm the charge, signs an authorization via your TEE-backed OKX Agentic Wallet, and replays the request.

Settlement happens on-chain on X Layer in 1-2 seconds, broker-fronted by OKX. The transaction hash is returned in the `payment-response` header — keep it as your receipt.

If the seller's handler returns a 4xx error (e.g. asset not available), **no payment is charged** — the broker only finalizes on success.

## Endpoints reference

See [`ethy-paid-data/SKILL.md`](./ethy-paid-data/SKILL.md) for the full endpoint catalog, pricing, and parameter conventions. TL;DR:

| Endpoint | Price | Returns |
|---|---|---|
| `/paid/v1/xlayer/signal/<chain>/<asset>` | $0.05 | Latest BUY/SELL signal with entry / stop / take-profit |
| `/paid/v1/xlayer/indicators/<chain>/<asset>` | $0.05 | Multi-TF technical indicators (RSI, MACD, EMA, BB, ADX) |
| `/paid/v1/xlayer/score/<chain>/<asset>` | $0.10 | Ethy Score 0–100 with component breakdown |
| `/paid/v1/xlayer/analysis/<chain>/<asset>` | $0.50 | LLM chart analysis — support, resistance, patterns, bias |

`<chain>` ∈ `base` · `xlayer` · `arbitrum` · `global` (for BTC/ETH price-only).
`<asset>` is a ticker or `0x`-prefixed contract address.

## Links

- [ethyai.app](https://ethyai.app) — Ethy AI app
- [`api.ethyai.app/paid/v1/xlayer/*`](https://api.ethyai.app) — Live paid API
- [OKX Agent Payments Protocol](https://web3.okx.com/onchainos/dev-docs/payments/overview) — Settlement layer
- [Skills npm package](https://www.npmjs.com/package/skills) — The installer
- [okx/onchainos-skills](https://github.com/okx/onchainos-skills) — Companion skill bundle for OKX Agentic Wallet + payment protocol

## License

MIT.
