# DLMM Meteora Analysis

Solana DLMM (Dynamic Liquidity Market Maker) analysis tools for Meteora pools.

## Features

- **VP Dry-Run Dashboard** — Compare simulated vs real on-chain data
- **Bin Range Accuracy** — Track position accuracy over time
- **Pool Monitoring** — Real-time active_bin tracking via Solana RPC
- **DLMM vs AMM Analysis** — Technical comparison and insights

## Files

| File | Description |
|------|-------------|
| `vp-dryrun-compare.html` | Interactive dashboard with embedded real data |
| `agent.md` | Complete analysis reference document |
| `dry-run-state.json` | Latest dry-run state data |

## Quick Start

1. Open `vp-dryrun-compare.html` in your browser
2. View simulated vs real bin comparison
3. No CORS issues — all data embedded!

## Technical Details

### Fetching Real Data

```javascript
// Solana RPC call
const response = await fetch("https://api.mainnet-beta.solana.com", {
  method: "POST",
  headers: {"Content-Type": "application/json"},
  body: JSON.stringify({
    jsonrpc: "2.0",
    id: 1,
    method: "getAccountInfo",
    params: [poolAddress, {encoding: "base64"}]
  })
});

// Decode active_bin from offset 48
const base64Data = response.result.value.data[0];
const raw = atob(base64Data);
const activeBin = raw.charCodeAt(48) | (raw.charCodeAt(49) << 8) |
                  (raw.charCodeAt(50) << 16) | (raw.charCodeAt(51) << 24);
```

### CORS Issue & Solution

Browser blocks Solana RPC from `file://` origin. Solution: Fetch data server-side and embed in HTML.

## Pool Addresses

| Pool | Address |
|------|---------|
| HENRY-SOL | `6eR5rRdexbht8aiiQmYq7yKb7EhdD3af22B4mHDmCp8x` |
| Magpie-SOL | `J9qgZAYeycmj5Ct9KmC8RfQZZDVzGwf5VfRoN4KNjnME` |
| Bountywork-SOL | `2C1XgnTarjmMNZpup44BL3rjsuWLPgsF3pHjzwRiXthS` |
| GACHA-SOL | `9bL8Pptpb8M2jEAb63EoarA6Po6Akarpha3JPzQfXGS3` |
| three-SOL | `8eDUNVrNUZ87SLYeihLTM7hJhckdAKqua6hPztuD57pX` |

## License

MIT
