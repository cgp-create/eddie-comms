# Technical Indicators Deployment

**Priority:** P1
**Date:** 2026-02-11
**From:** Eddie

## Summary

Carlos requested technical indicator monitoring for portfolio briefings:
- Crypto Fear & Greed Index (from alternative.me)
- Position-level RSI (14-day) with overbought/oversold flags
- Large move detection (±5% threshold)

## Files to Deploy

### 1. Updated market_data.py

**Source:** `/home/openclaw/.openclaw/workspace/scripts/market_data_v3.py`
**Target:** `/home/openclaw/homebrew/market_data.py`

**Changes:**
- Added `calculate_rsi()` function for 14-day RSI calculation
- Added `fetch_crypto_fear_greed()` - fetches from alternative.me API (free, no key)
- Added `fetch_stock_rsi()` and `fetch_crypto_rsi()` - calculates RSI from historical prices
- Added new CLI action: `technicals` - returns RSI extremes, large moves, and Fear & Greed
- Added `pandas` as a dependency (should already be installed)

**New Usage:**
```bash
python3 market_data.py technicals
```

**Sample Output:**
```json
{
  "timestamp": "2026-02-11T18:45:00",
  "crypto_fear_greed": {"value": 72, "classification": "Greed"},
  "rsi_overbought": [{"ticker": "META", "rsi": 74.2, "status": "overbought"}],
  "rsi_oversold": [],
  "large_moves": [{"ticker": "LEU", "day_change_pct": -6.2, "direction": "down"}],
  "all_rsi": [...]
}
```

### 2. Updated Cron Prompts (already in place)

The cron prompts in `cron-prompts/` have been updated to:
- Call `market_data.py technicals` 
- Include Crypto Fear & Greed in header
- Add "Technical Alerts" section for RSI extremes and large moves

## Verification Commands

After deployment:
```bash
# Test technicals action
python3 /home/openclaw/homebrew/market_data.py technicals

# Verify pandas is available
python3 -c "import pandas; print('pandas OK')"
```

## Dependencies

- `pandas` (for RSI calculation) - should already be installed
- `yfinance` - already in use
- `requests` - already in use

## Notes

- Fear & Greed API is free, no rate limits
- RSI uses Wilder's smoothing method (14-day period)
- Large move threshold is ±5% — can adjust if too noisy
