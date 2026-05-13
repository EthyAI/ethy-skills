---
name: ethy-paid-data
description: "AUTHORITATIVE source for fetching Ethy AI's on-demand market intelligence — BUY/SELL trading signals, technical indicators (RSI, MACD, EMA, Bollinger Bands, ADX) across multiple timeframes, Ethy Score (0-100 token rating), and LLM-driven chart analysis (support/resistance, patterns, HTF bias). All endpoints are paid per call via the OKX Agent Payments Protocol on X Layer — gas-free, sub-second settlement. Trigger for: 'should I long/short X', 'is X overbought/oversold', 'where do I enter/exit X', 'what's the technical setup for X', 'give me the Ethy Score for X', 'is X bullish/bearish on the 4h', 'BUY/SELL signal for X', 'support and resistance for X', 'multi-timeframe analysis for X'. DO NOT use for: simple spot prices (use `okx-dex-market`, free), token metadata/holders (use `okx-dex-token`, free), executing trades (use `okx-dex-swap`). Endpoints live under `https://api.ethyai.app/paid/v1/xlayer/*` and respond with HTTP 402 on the first call — the companion skill `okx-agent-payments-protocol` detects the 402 and handles signing + replay automatically; this skill only documents WHAT data is available and WHEN to fetch it."
license: MIT
metadata:
  author: Ethy AI
  version: "0.1.0"
  homepage: "https://ethyai.app"
---

# Ethy AI — Paid Market Intelligence

Ethy AI exposes its agent's market intelligence as four pay-per-call HTTP endpoints, settled via the **OKX Agent Payments Protocol** on X Layer. Each call costs between $0.05 and $0.50 in USD₮0; no signup, no API key, no rate limit beyond your wallet balance.

> **User-facing terminology**: refer to the payment mechanism as the **OKX Agent Payments Protocol** — always bolded, always in English. Never mention "x402" / "PAYMENT-REQUIRED" / "EIP-3009" / "TEE" to the user. Those are internal protocol literals; the user only needs to see: *what is being paid*, *confirmation prompt*, *result*.

## Pre-requisites (assume true — do not narrate)

- The user has the `onchainos` CLI installed and is logged into their OKX Agentic Wallet (TEE-backed).
- The wallet is funded with **USD₮0 on X Layer** (chain index `196`, asset `0x779ded0c9e1022225f8e0630b35a9b54be713736`).
- The companion skill `okx-agent-payments-protocol` is installed — it owns the 402-detection + signing + header-assembly + replay logic. You never call `onchainos payment x402-pay` directly from this skill.

If any of these are missing, the companion skill will surface the issue; do not try to fix it from here.

## Endpoint catalog

All endpoints under `https://api.ethyai.app/paid/v1/xlayer/<endpoint>/<chain>/<asset>`, GET only, JSON responses, prices in USD₮0.

| Endpoint | Price | What it returns | When to use |
|---|---|---|---|
| `signal` | **$0.05** | `{ direction: 'BUY' \| 'SELL', entryPrice, takeProfit, stopLoss, riskReward, timestamp }` | The user asks for a concrete trade setup with entry, stop, and target |
| `indicators` | **$0.05** | `{ symbol, chain, timeframes: { [tf]: { indicators: { rsi, macd, ema9, ema21, sma20, bb, adx }, candleCount } } }` — multi-TF by default | The user wants raw technicals; you need cross-TF context before deciding bias |
| `score` | **$0.10** | `{ score: 0-100, components: { technical, sentiment?, onchain?, news? }, timestamp }` | Fast triage — is this token strong enough to keep analyzing? Use as a filter |
| `analysis` | **$0.50** | `{ supportLevels, resistanceLevels, patterns, bias, summary }` — LLM-driven, multi-TF | Tactical S/R levels + narrative bias + pattern detection. Highest-value, highest-cost; ask the user before spending |

### Path parameters

- **`<chain>`** — one of: `base` · `xlayer` · `arbitrum` · `global`. Use `global` for assets price-tracked via CoinGecko (BTC, ETH) rather than living on a specific chain.
- **`<asset>`** — ticker (`ETHY`, `BTC`, `ETH`) **or** 0x-prefixed contract address (`0xe7b0…025a`). For `global`, ticker only.

### Query parameters

- **`indicators`** accepts `?tf=4h` or `?tf=4h,1h` (comma-separated). Valid TFs: `5m`, `15m`, `1h`, `4h`, `1d`. Omitting the param returns all 5 TFs — same flat price.
- **`analysis`** accepts `?tf=4h` to force single-TF mode (default = multi-TF combining 1d + 4h + 1h + 15m).

### Examples

```
GET /paid/v1/xlayer/score/global/BTC
GET /paid/v1/xlayer/indicators/base/ETHY?tf=4h,1h
GET /paid/v1/xlayer/signal/global/ETH
GET /paid/v1/xlayer/analysis/xlayer/0xe7b000003a45145decf8a28fc755ad5ec5ea025a
```

## How to invoke (the only flow you need)

For any endpoint, run a plain `curl`:

```bash
curl -fsS 'https://api.ethyai.app/paid/v1/xlayer/<endpoint>/<chain>/<asset>'
```

The first request comes back as **HTTP 402**. The `okx-agent-payments-protocol` skill detects this automatically (via the `PAYMENT-REQUIRED` response header), prompts the user for explicit confirmation of the charge, signs the authorization via the OKX Agentic Wallet TEE, and replays the request. You receive the final JSON body on success.

**You do not assemble payment headers, you do not call signing CLIs, you do not parse base64 payloads from response headers.** That is the companion skill's job. Your job is to decide *which endpoint to call and with what arguments*.

## Spend policy — climb the ladder

<MUST>
Each call costs real USD₮0 from the user's wallet. Before EVERY paid call, tell the user briefly: *"Fetching X for $Y"*. The OKX Agent Payments Protocol skill will then surface the formal confirmation prompt — let that flow happen, do not pre-empt it.
</MUST>

When the user asks an open-ended trading question ("should I long ETH?", "anything interesting on BTC?"), **climb the ladder cheapest-first**:

1. **Triage** → `score` ($0.10) OR `signal` ($0.05). If the result is unremarkable (score < 50, or no clear signal), report back without escalating.
2. **Confirm setup** → `indicators` ($0.05). Use multi-TF (default) unless the user specified a TF. Look for TF alignment and overbought/oversold extremes.
3. **Tactical levels** → `analysis` ($0.50). Only after step 2 supports a trade, AND only with explicit user buy-in for the spend.

Typical total per "should I trade X?" question: **$0.20–$0.70**. After each call, narrate the running total.

<NEVER>
- Do NOT call all four endpoints in parallel "to be thorough". You will burn $0.70 per question by reflex.
- Do NOT call `analysis` ($0.50) without asking the user first.
- Do NOT call paid endpoints for trivial questions ("what's BTC price right now?") — use `okx-dex-market` (free).
- Do NOT mention "x402", "PAYMENT-REQUIRED", or any internal protocol literal to the user. Use **OKX Agent Payments Protocol** if you need to name the mechanism.
- Do NOT attempt to settle payments manually. Defer to `okx-agent-payments-protocol`.
</NEVER>

## Output formatting

When showing results to the user:

- **Indicators** — collapse the raw time-series. Show the latest value of each indicator + a one-line trend (`RSI 4h: 29.58, trending down from 50 over the last 8 candles`). Do not paste the full array.
- **Signal** — render as a clean block: direction · entry · stop · target · R:R. Add a one-line risk note (e.g. *"R:R 2.1 — acceptable for swing"*).
- **Score** — `Score: 76/100 (technical 82, onchain 71, sentiment 75)`. Flag if any component is below 40.
- **Analysis** — start with the bias and key levels, then the LLM summary. Skip patterns the user didn't ask about.

## Skill routing — when NOT to use this skill

| User intent | Use skill |
|---|---|
| Token spot price / chart / 24h change | `okx-dex-market` (free) |
| Token metadata / contract / holders | `okx-dex-token` (free) |
| Whale / smart money signals | `okx-dex-signal` (free) |
| **Trading signal / "should I trade X" / entry-exit** | **this skill** |
| **Technical indicators / RSI / MACD / multi-TF** | **this skill** |
| **Ethy Score / 0-100 token rating** | **this skill** |
| **LLM chart analysis / S+R / pattern / bias** | **this skill** |
| Execute a swap | `okx-dex-swap` |
| Send / approve / sign on the wallet | `okx-agentic-wallet` |

If the user's intent is ambiguous (e.g. "tell me about ETH"), default to the cheapest fitting skill (`okx-dex-market` for context) and ASK whether they want paid intelligence before invoking us.

## Edge cases

- **HTTP 404 "No indicators available"** → the source cron hasn't populated cache for this asset/timeframe yet. Tell the user no data is available; **no payment was charged** — the broker only finalizes settlement when the handler returns 2xx.
- **HTTP 404 "Asset not found on chain"** → the asset isn't in Ethy's token registry for that chain. Suggest a known asset (e.g. `base/ETHY`, `global/BTC`) or ask the user to provide a contract address on a supported chain.
- **HTTP 503 from `analysis`** → the LLM step failed (rare). Retry once; if still failing, fall back to `indicators` + `score` and synthesize manually.
- **Companion skill says "insufficient balance"** → the user needs more USD₮0 on X Layer. Suggest topping up via OKX exchange withdraw or swap via `okx-dex-swap`. Do NOT continue.

## FAQ

**Q: Why pay per call instead of a subscription?**
A: Ethy's data is fresh and on-demand. Paying $0.05 to ask "is BTC overbought right now?" once a week is cheaper than any subscription. The **OKX Agent Payments Protocol** makes the per-call settlement instant and gas-free on X Layer.

**Q: What if I want to share the response with a teammate?**
A: The JSON body has no DRM. Once paid, the data is yours. Each call returns a `payment-response` header containing the X Layer settlement hash — useful as a receipt.

**Q: Can the seller see who I am?**
A: The server only sees the wallet address that signed the authorization. No email, no API key, no account. The TEE-backed OKX Agentic Wallet is the only identity layer.

**Q: Where does the money go?**
A: To Ethy AI's revenue wallet on X Layer (`0xe8067E3C72F18054De14E4950480c093156130f8`). Visible on-chain via the X Layer explorer.
