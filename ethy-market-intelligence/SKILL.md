---
name: ethy-market-intelligence
description: "AUTHORITATIVE source for fetching Ethy AI's on-demand market intelligence — BUY/SELL trading signals, technical indicators (RSI, MACD, EMA, Bollinger Bands, ADX) across multiple timeframes, Ethy Score (0-100 token rating), and LLM-driven chart analysis (support/resistance, patterns, HTF bias). All endpoints are paid per call via the OKX Agent Payments Protocol on X Layer — gas-free, sub-second settlement. Trigger for: 'should I long/short X', 'is X overbought/oversold', 'where do I enter/exit X', 'what's the technical setup for X', 'give me the Ethy Score for X', 'is X bullish/bearish on the 4h', 'BUY/SELL signal for X', 'support and resistance for X', 'multi-timeframe analysis for X'. DO NOT use for: simple spot prices (use `okx-dex-market`, free), token metadata/holders (use `okx-dex-token`, free), executing trades (use `okx-dex-swap`). Endpoints live under `https://api.ethyai.app/paid/v1/xlayer/*` and respond with HTTP 402 on the first call — the companion skill `okx-agent-payments-protocol` detects the 402 and handles signing + replay automatically; this skill only documents WHAT data is available and WHEN to fetch it."
license: MIT
metadata:
  author: Ethy AI
  version: "0.2.0"
  homepage: "https://ethyai.app"
---

# Ethy AI — Market Intelligence

Ethy AI exposes its agent's market intelligence as four pay-per-call HTTP endpoints, settled via the **OKX Agent Payments Protocol** on X Layer. Each call costs between $0.05 and $0.25 in USD₮0; no signup, no API key, no rate limit beyond your wallet balance.

## Required companion skills

This skill only documents **what data to fetch and when**. The actual payment plumbing lives elsewhere — without the companion skills below installed, the 402 response cannot be settled and the buyer never sees a result.

| Companion skill | Why it's required |
|---|---|
| **`okx-agentic-wallet`** | The TEE-backed wallet that signs the EIP-3009 authorization. Must be logged in and funded with USD₮0 on X Layer. |
| **`okx-agent-payments-protocol`** | Detects HTTP 402 on the response, decodes the challenge, walks the user through the confirmation prompt, signs via the agentic wallet, and replays the request. Owns the entire 402 → pay → retry round trip. |

Both ship together as part of the OnchainOS skills bundle (`npx skills add okx/onchainos-skills`). If either is missing, fall back to telling the user "install OnchainOS skills first" — do not attempt to assemble payment headers manually from this skill.

## How you behave when this skill activates

<MUST>
**Do NOT preamble.** When the user asks something this skill covers (e.g. "ask Ethy to analyze WOKB", "is ETH overbought?"), go DIRECTLY to the action — never announce "I found the Ethy skill" or "I'll use the market-intelligence skill". Just say what you're about to fetch and let the payment flow happen.

**Do announce the payment mechanism, every time, before each paid call.** Use this template verbatim (translated to the user's language):

> *"This is a paid endpoint. I'll fetch `<endpoint>` for **`<asset>`** for **$X.XX USD₮0** via the **OKX Agent Payments Protocol** on X Layer. The confirmation prompt will appear in a moment — review and confirm to proceed."*

After that line, run the `curl`. The companion skill `okx-agent-payments-protocol` takes over with its own branded confirmation prompt. Do NOT step on that prompt — it's the source of truth for the actual yes/no.
</MUST>

> **User-facing terminology**: always say **OKX Agent Payments Protocol** (bolded, English). Never expose "x402" / "PAYMENT-REQUIRED" / "EIP-3009" / "TEE" to the user. Those are internal protocol literals; the user only sees: *what is being paid*, *confirmation*, *result*.

## Endpoint catalog

All endpoints under `https://api.ethyai.app/paid/v1/xlayer/<endpoint>/<chain>/<asset>`, GET only, JSON responses, prices in USD₮0.

| Endpoint | Price | What it returns | When to use |
|---|---|---|---|
| `signal` | **$0.05** | `{ direction: 'BUY' \| 'SELL', entryPrice, takeProfit, stopLoss, riskReward, rationale, symbol, chain, timeframe, timestamp }` | The user asks for a concrete trade setup with entry, stop, and target |
| `indicators` | **$0.05** | `{ symbol, chain, timeframes: { [tf]: { indicators: { rsi, macd, ema9, ema21, sma20, bb }, candleCount } } }` — multi-TF by default | The user wants raw technicals; you need cross-TF context before deciding bias |
| `score` | **$0.10** | `{ score: 0-100, components: { technical, momentum, volatility, structure }, rationale, asset, chain, timestamp }` | Fast triage — is this token strong enough to keep analyzing? Use as a filter |
| `analysis` | **$0.25** | `{ asset, timeframe, supportLevels, resistanceLevels, patterns, summary, bias, analyzedAt, chain }` — LLM-driven, multi-TF | Tactical S/R levels + narrative bias + pattern detection. Ask the user before spending |

### Path parameters

- **`<chain>`** — one of: `base` · `xlayer` · `arbitrum`. The chain where the asset lives in Ethy's token registry.
- **`<asset>`** — ticker (`ETHY`, `WOKB`, `VIRTUAL`, `AERO`) **or** 0x-prefixed contract address (`0xe7b0…025a`). Use the chain-specific symbol or contract; cross-chain assets like BTC should be requested via their wrapped form on a real chain (e.g. `base/cbBTC`).

### Query parameters

- **`indicators`** accepts `?tf=4h` or `?tf=4h,1h` (comma-separated). Valid TFs: `5m`, `15m`, `1h`, `4h`, `1d`. Omitting the param returns all 5 TFs — same flat price.
- **`analysis`** accepts `?tf=4h` to force single-TF mode (default = multi-TF combining 1d + 4h + 1h + 15m).

### Examples

```
GET /paid/v1/xlayer/indicators/xlayer/WOKB
GET /paid/v1/xlayer/indicators/base/ETHY?tf=4h,1h
GET /paid/v1/xlayer/analysis/base/VIRTUAL
GET /paid/v1/xlayer/analysis/xlayer/0xe7b000003a45145decf8a28fc755ad5ec5ea025a
```

## The 402 → pay → retry round trip

Every paid call follows the same three-step dance, and the companion skill `okx-agent-payments-protocol` automates all of it. **From this skill's point of view, all you do is `curl` — the companion handles the rest.** The mechanics are documented here for clarity:

### Step 1 — initial GET returns 402

```bash
curl -i 'https://api.ethyai.app/paid/v1/xlayer/signal/xlayer/WOKB'
```

Response:
```
HTTP/2 402
content-type: application/json
payment-required: eyJ4NDAyVmVyc2lvbiI6MiwiZXJyb3IiOiJQYXltZW50IHJlcXVpcmVkIi…

{}
```

The `payment-required` header is a base64-encoded JSON challenge:
```json
{
  "x402Version": 2,
  "resource": { "url": "...", "description": "Latest trading signal…" },
  "accepts": [{
    "scheme": "exact",
    "network": "eip155:196",
    "amount": "50000",
    "asset": "0x779ded0c9e1022225f8e0630b35a9b54be713736",
    "payTo": "0xe8067E3C72F18054De14E4950480c093156130f8",
    "maxTimeoutSeconds": 300,
    "extra": { "name": "USD₮0", "version": "1" }
  }]
}
```

### Step 2 — sign the authorization

The companion skill calls the agentic wallet's TEE signer (under the hood: `onchainos payment x402-pay --network eip155:196 --amount 50000 --pay-to 0xe806… --asset 0x779d…`) and produces an EIP-3009 `transferWithAuthorization` signature. The user sees the OnchainOS branded confirmation prompt with the exact amount + payee + network before the signature is generated.

### Step 3 — replay the request with the `PAYMENT-SIGNATURE` header

The companion skill assembles the final header:
```
PAYMENT-SIGNATURE: <base64 of { x402Version, resource, accepted, payload: { signature, authorization } }>
```

…and replays:
```bash
curl -i -H "PAYMENT-SIGNATURE: <header>" 'https://api.ethyai.app/paid/v1/xlayer/signal/xlayer/WOKB'
```

On success:
```
HTTP/2 200
content-type: application/json
payment-response: <base64 with the X Layer settlement tx hash>

{ "direction": "BUY", "entryPrice": 85.91, "takeProfit": 89.19, ... }
```

The `payment-response` header decodes to `{ network, payer, success: true, transaction: "0x…" }` — that's your on-chain receipt.

### What you actually do from this skill

```bash
curl -fsS 'https://api.ethyai.app/paid/v1/xlayer/<endpoint>/<chain>/<asset>'
```

That's it. The 402 hits the companion skill, the user confirms, the replay happens, and you receive the 200 body. **You do not assemble payment headers, you do not call signing CLIs, you do not parse base64 payloads.** That is the companion skill's job. Your job is to decide *which endpoint to call and with what arguments*, then format the result.

## Spend policy — climb the ladder

When the user asks an open-ended trading question ("should I long ETH?", "anything interesting on BTC?"), **climb the ladder cheapest-first**, one endpoint at a time:

1. **Triage** → `score` ($0.10) OR `signal` ($0.05). If the result is unremarkable (score < 50, or no clear signal), report back without escalating.
2. **Confirm setup** → `indicators` ($0.05). Use multi-TF (default) unless the user specified a TF. Look for TF alignment and overbought/oversold extremes.
3. **Tactical levels** → `analysis` ($0.25). Only after step 2 supports a trade, AND only with explicit user buy-in for the spend.

Typical total per "should I trade X?" question: **$0.15–$0.45**. After each call, **narrate the running total** in a compact footer (see "Output formatting").

When the user requests a **complete analysis explicitly** (e.g. "full analysis of WOKB", "complete report on X"), it is acceptable to call `analysis` first — but **announce the higher cost upfront and confirm** before invoking.

<NEVER>
- Do NOT call all four endpoints in parallel "to be thorough". You will burn $0.45 per question by reflex.
- Do NOT call `analysis` ($0.25) without asking the user first.
- Do NOT call paid endpoints for trivial questions ("what's BTC price right now?") — use `okx-dex-market` (free).
- Do NOT mention "x402", "PAYMENT-REQUIRED", or any internal protocol literal to the user. Use **OKX Agent Payments Protocol** if you need to name the mechanism.
- Do NOT attempt to settle payments manually. Defer to `okx-agent-payments-protocol`.
</NEVER>

## Output formatting

Render every endpoint result with a clear ASCII-bordered block + a one-line footer with the running cost. Claude Code's TUI renders markdown tables — use those when they fit, fall back to the box format below for compact single-record outputs.

### Trading Signal — output template

```
╭─ Trading Signal · <asset> ───────────╮
│  Direction      <BUY | SELL>          │
│  Entry          $<price>              │
│  Take profit    $<price>  (<±X.X%>)   │
│  Stop loss      $<price>  (<±X.X%>)   │
│  Risk / Reward  <X.X>                 │
╰───────────────────────────────────────╯
```
Add a one-line note assessing the R:R quality (e.g. *"R:R 2.0 — acceptable for swing positions"*). If the response includes `rationale`, append it underneath.

### Multi-TF Indicators — output template

Always present as a markdown table. Collapse the raw time-series to the **latest value** of each indicator per TF; never paste the full array:

```
| TF    | RSI  | MACD signal | EMA9 vs EMA21 | Trend |
|-------|------|-------------|---------------|-------|
| 5m    | 38.2 | bullish     | above         | ⤴     |
| 15m   | 42.1 | bullish     | above         | ⤴     |
| 1h    | 51.4 | neutral     | flat          | →     |
| 4h    | 55.0 | bullish     | above         | ⤴     |
| 1d    | 62.1 | bullish     | above         | ⤴     |
```
After the table, add 1-2 lines of plain interpretation (TF alignment, divergences, overbought/oversold flags).

### Ethy Score — output template

```
╭─ Ethy Score · <asset> ───────────────╮
│  Score        <0-100> / 100           │
│  Technical    <0-100>                 │
│  Momentum     <0-100>                 │
│  Volatility   <0-100>                 │
│  Structure    <0-100>                 │
╰───────────────────────────────────────╯
```
Flag any component < 40 with an inline note. If the response includes `rationale`, append it underneath.

### AI-powered Chart Analysis — output template

```
╭─ AI-powered Analysis · <asset> ──────╮
│  Bias         <long | short | neutral>│
│  Support      $<p1> · $<p2> · $<p3>   │
│  Resistance   $<p1> · $<p2> · $<p3>   │
│  Pattern      <short description>      │
╰───────────────────────────────────────╯

<2-3 sentence summary from the LLM, edited for brevity if needed>
```
Use the phrase **"AI-powered analysis"** when referring to this endpoint to the user — never "LLM analysis".

### Running cost footer — always include

After each paid call, append a single-line footer:

```
💸 Paid $X.XX USD₮0 · settled tx <0xshort…tail> · running total $Y.YY · X Layer · gas $0
```

Show the X Layer settlement hash (short form, e.g. `0xc435…597bf`) — the `payment-response` header returned by every successful call contains the full hash.

## Edge cases

- **HTTP 404 "Not enough candles to generate a signal"** / **"Not enough candles to score"** → the asset is in the registry but CoinGecko didn't return enough recent OHLCV. **No payment was charged** — the broker only finalizes settlement when the handler returns 2xx.
- **HTTP 404 "Asset not found on chain"** → the asset isn't in Ethy's token registry for that chain. Suggest a known asset (e.g. `base/ETHY`, `xlayer/WOKB`, `base/VIRTUAL`, `base/AERO`) or ask the user to provide a contract address on a supported chain.
- **HTTP 503 from `analysis` / `signal` / `score`** → the LLM step failed (rare). Retry once; if still failing, surface the error to the user.
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
