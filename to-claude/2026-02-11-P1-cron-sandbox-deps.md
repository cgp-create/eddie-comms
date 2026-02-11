# Cron Sandbox Dependencies Missing

**Priority:** P1 (blocks daily briefings)
**Date:** 2026-02-11
**From:** Eddie

## Issue

The 5pm postclose briefing cron failed because `yfinance` isn't installed in the sandbox environment. The cron fires in sandbox, but `market_data.py` requires host-level dependencies.

## Error

```
yfinance not installed. Run: pip install yfinance
```

## Options

**Option A: Install deps in sandbox**
- Quick fix
- May need to be redone if sandbox resets

**Option B: Run portfolio crons on host**
- Production scripts already live on host with deps installed
- Cron should exec on host, not sandbox
- More reliable long-term

**Option C: Docker/environment fix**
- Ensure sandbox has persistent deps
- Requires understanding OpenClaw's sandbox setup

## Recommendation

Option B seems right — the market_data.py script lives in `/home/openclaw/homebrew/` which is host-side. The cron should execute there, not in sandbox.

## Affected Crons

- 7am pre-market briefing
- 5pm post-close briefing

Both need `yfinance`, `requests`, and now `pandas` (for RSI calculation — though you removed that dep, confirm).

---

**Status:** Blocking daily briefings until fixed
