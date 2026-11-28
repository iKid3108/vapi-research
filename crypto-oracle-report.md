# Crypto Oracle API — Accuracy Test Report

Task: vAPI work order `135b13d2-ed20-4ec9-8765-c5a0a9f4e8c4` ("Test my Crypto Oracle API and report accuracy").
Tester: linked vAPI agent wallet `0x9E63…52b7`. Test date: 2026-11-28 ~05:05 UTC.

## 1. Results

### bitcoin — SUCCESS
- Called: `GET /price` (no query → server default coin bitcoin), paid via `vapi pay` ($0.001 USDC, Base, settled OK).
- Oracle response: `price_usd = 59,915.74`, `change_24h_percent = -2.3188`, `source: CoinGecko`, `as_of: 2026-11-28T05:00:10Z`.
- Response time (client-observed, incl. x402 verify+settle roundtrip via CLI): **~800 ms**.
- CoinGecko comparison (same minute): **$59,945** → difference **0.05%** → **MATCH (within 1%)**. 24h change: oracle -2.32% vs CoinGecko -2.29% → consistent.

### ethereum — BLOCKED (test could not complete)
- `GET /price?coin=ethereum` (and `?coin=solana`): the server correctly returns a valid x402 v2 402 offer ($0.001, payTo `0x0836…CF07`, verified manually with curl).
- However, paying through `vapi-cli pay` with a **query string in the URL fails**: the CLI returns `{"status":404,"body":{"error":"not found"},"payment":null}` — no payment is attempted and no API data is returned. The same command without a query string works (see bitcoin).
- POST with `{"coin":"ethereum"}` body also returns 404 from the pay flow.
- **Issue found:** query-string URLs are not usable through the `vapi pay <id-or-url>` flow for this API (CLI 0.15.1). Payment was not sent for these calls, so no funds were wasted.

### solana — BLOCKED (same cause as ethereum)

## 2. Summary
| Coin | Oracle price | CoinGecko | Match ≤1%? | Response time | Paid? |
|---|---|---|---|---|---|
| bitcoin | 59,915.74 | 59,945 | ✅ 0.05% | ~800 ms | ✅ $0.001 |
| ethereum | — (pay-flow blocked) | 1,661.79 | — | — | ❌ not charged |
| solana | — (pay-flow blocked) | 71.55 | — | — | ❌ not charged |

## 3. Issues & recommendations
1. **Query parameters break `vapi pay`** for this endpoint (404 "not found", no payment attempted). Suggestion for the API owner: add path-based variants (e.g. `/price/bitcoin`) or accept the coin via POST body, so agents using CLI flows can pay per coin. (A CLI-side fix would also resolve it.)
2. **Nonexistent coins are accepted** (`?coin=foo` returns a normal 402 offer instead of a 4xx) — harmless here since no payment is taken before validation, but a pre-validation of the coin id would be cleaner.
3. Price accuracy for the one testable coin is excellent: **0.05% deviation from CoinGecko**, well inside the 1% threshold, with matching 24h change.

## Sources
- Oracle API: https://crypto-oracle.akuncuanarypto.workers.dev/price
- CoinGecko comparison: https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum,solana&vs_currencies=usd&include_24hr_change=true
- vAPI pay flow: https://docs.vapinetwork.ai/how-vapi-works.md
