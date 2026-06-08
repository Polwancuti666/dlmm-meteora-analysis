# VP Dry-Run Analysis — Solana DLMM Bot

## Overview
Analysis of Virtual Position (VP) dry-run data from a Meteora DLMM bot on Solana. The bot runs paper trading (virtual positions) without actual on-chain transactions.

**Data Sources:**
- Simulated data: dry-run-state.json (bot's internal state)
- Real data: Solana RPC (`getAccountInfo` → decode pool account, offset 48 = active_bin)

---

## Pool Addresses & Real-Time Data (2026-06-07)

| Pool | Address | Real Active Bin | Bin Step |
|------|---------|-----------------|----------|
| HENRY-SOL | `6eR5rRdexbht8aiiQmYq7yKb7EhdD3af22B4mHDmCp8x` | -486 | 100 |
| Magpie-SOL | `J9qgZAYeycmj5Ct9KmC8RfQZZDVzGwf5VfRoN4KNjnME` | -479 | 80 |
| Bountywork-SOL | `2C1XgnTarjmMNZpup44BL3rjsuWLPgsF3pHjzwRiXthS` | -485 | 100 |
| GACHA-SOL | `9bL8Pptpb8M2jEAb63EoarA6Po6Akarpha3JPzQfXGS3` | -463 | 100 |
| three-SOL | `8eDUNVrNUZ87SLYeihLTM7hJhckdAKqua6hPztuD57pX` | -317 | 80 |

---

## Bin Range Accuracy Check

### Accuracy Results

| Pool | Simulated | Real | Diff | In Range | Status |
|------|-----------|------|------|----------|--------|
| HENRY-SOL | -478 | -486 | -8 | ✅ Yes | ⚠️ Small deviation |
| Magpie-SOL | -526 | -479 | +47 | ❌ No | ❌ Big deviation |
| Bountywork-SOL | -489 | -485 | +4 | ✅ Yes | ✅ Accurate |
| GACHA-SOL | -465 | -463 | +2 | ✅ Yes | ✅ Accurate |
| three-SOL | -302 | -317 | -15 | ✅ Yes | ⚠️ Medium deviation |

### Summary
- **In range:** 3/5 (60%)
- **Accurate (diff ≤ 5):** 2/5 (40%)
- **Average diff:** 15.2 bins

### Key Findings
- Bountywork-SOL & GACHA-SOL are highly accurate
- HENRY-SOL: Small deviation (-8 bins), price moved slightly
- three-SOL: Medium deviation (-15 bins), relatively new pool
- Magpie-SOL: Big deviation (+47 bins), likely pool recycled or significant price movement

---

## DLMM vs AMM on Solana

### AMM (Automated Market Maker)
**Examples:** Raydium, Orca, Jupiter

- Formula: `x * y = k` (Constant Product)
- Liquidity spread uniformly from price 0 to ∞
- Capital efficiency: Low (~5-10%)
- Fees: Fixed (typically 0.25-0.3%)
- Impermanent loss: Higher
- Complexity: Low

### DLMM (Dynamic Liquidity Market Maker)
**Examples:** Meteora DLMM

- Formula: Liquidity per BIN (price range)
- Liquidity concentrated in specific price ranges
- Capital efficiency: High (~50-100%)
- Fees: Dynamic (0.01-10% based on volatility)
- Impermanent loss: Lower
- Complexity: Higher

### Comparison Table

| Aspect | AMM | DLMM |
|--------|-----|------|
| Structure | Constant curve (x*y=k) | Bin/bin-step |
| Liquidity | Infinite range | Specific range |
| Capital Efficiency | Low (~5-10%) | High (~50-100%) |
| Fee | Fixed (0.25%) | Dynamic (0.01-10%) |
| Impermanent Loss | Higher | Lower |
| Complexity | Low | High |

### Why DLMM is More Efficient
```
AMM:  $1000 spread across $0-$∞
      → Effective capital: ~$50

DLMM: $1000 in range $95-$105
      → Effective capital: ~$500

Efficiency: 10x better!
```

### DLMM Advantages for Bot Trading
1. Can set specific price ranges (as implemented)
2. Higher fees during high volatility
3. Less impermanent loss
4. Real-time active_bin monitoring possible

---

## Solana RPC Technical Details

### Fetching Pool Account Data
```javascript
// POST to Solana RPC
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "getAccountInfo",
  "params": ["<pool_address>", {"encoding": "base64"}]
}

// Decode active_bin from response
const raw = atob(base64Data);
const activeBin = raw.charCodeAt(48) | (raw.charCodeAt(49) << 8) |
                  (raw.charCodeAt(50) << 16) | (raw.charCodeAt(51) << 24);
```

### CORS Issues with Local Files
- Browser blocks Solana RPC requests from `file://` origin
- **Solution:** Fetch data server-side (VPS), embed in HTML
- Solana RPC returns `Access-Control-Allow-Origin: backend_traffic` (not `*`)
- Alternative RPCs (Alchemy) have rate limits (429)

---

## VP Dry-Run Dashboard Files

| File | Description |
|------|-------------|
| `/root/vp-dryrun-compare.html` | Main dashboard with embedded real data |
| `/root/vp-archive-viz.html` | VP archive visualization |
| `/root/dry-run-state.json` | Latest dry-run state (5 positions) |
| `/root/vp-archive-2026-06.json` | VP archive data |

### Dashboard Features
1. **KPI Cards** — 7 metrics: positions, capital, PnL, peak PnL, snapshots, OOR, real data
2. **Comparison Cards** — Per-pool detail with accuracy gauge & bin visualization
3. **PnL Timeline Chart** — All 5 positions in 1 chart, different colors
4. **Bin Comparison Chart** — Visual bar: simulated (purple) vs real (cyan)
5. **Snapshot History** — Expandable, 10 latest snapshots per position
6. **Fetch Button** — Solana RPC direct (CORS-safe when embedded)

---

## Position Details

### HENRY-SOL
- Pool: `6eR5rRdexbht8aiiQmYq7yKb7EhdD3af22B4mHDmCp8x`
- Capital: 0.5 SOL (~$31)
- Range: [-518, -464], Step: 100
- Peak PnL: +5.61%
- OOR: 0 min
- Snapshots: 33

### Magpie-SOL
- Pool: `J9qgZAYeycmj5Ct9KmC8RfQZZDVzGwf5VfRoN4KNjnME`
- Capital: 0.5 SOL (~$31)
- Range: [-566, -516], Step: 80
- Peak PnL: +3.30%
- OOR: 0 min
- Snapshots: 11

### Bountywork-SOL
- Pool: `2C1XgnTarjmMNZpup44BL3rjsuWLPgsF3pHjzwRiXthS`
- Capital: 0.5 SOL (~$31)
- Range: [-550, -481], Step: 100
- Peak PnL: +5.67%
- OOR: 0 min
- Snapshots: 8

### GACHA-SOL
- Pool: `9bL8Pptpb8M2jEAb63EoarA6Po6Akarpha3JPzQfXGS3`
- Capital: 0.5 SOL (~$31)
- Range: [-534, -469], Step: 100
- Peak PnL: +0.53%
- OOR: 10 min
- Snapshots: 7

### three-SOL
- Pool: `8eDUNVrNUZ87SLYeihLTM7hJhckdAKqua6hPztuD57pX`
- Capital: 0.5 SOL (~$31)
- Range: [-344, -297], Step: 80
- Peak PnL: 0%
- OOR: 0 min
- Snapshots: 2

---

## Recommendations for Production

1. **Update bin ranges in real-time** — Current ranges are static
2. **Add tolerance threshold** — Alert if diff > ±10 bins
3. **Monitor OOR positions** — GACHA-SOL had 10 min OOR
4. **Consider pool recycling** — Verify pool address before trading
5. **Use DLMM advantages** — Dynamic fees, concentrated liquidity

---

## Technical Notes

### Meteora DLMM API
- Endpoint: `https://dlmm-api.meteora.ag/pair/{address}`
- Returns 404 for some pools (may be rate limited or deprecated)
- CORS blocked from local files

### Solana RPC
- Mainnet: `https://api.mainnet-beta.solana.com`
- CORS: `Access-Control-Allow-Origin: backend_traffic` (not `*`)
- Works from server-side, blocked from local `file://`
- Pool account size: 904 bytes

### Data Freshness
- Real data fetched: 2026-06-07T08:22:00Z
- Last snapshot: 2026-06-07T06:20:01.861Z
- SOL price at last snapshot: $64.42
