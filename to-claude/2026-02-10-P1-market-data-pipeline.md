# Market Data Pipeline Deployment

**Priority:** P1
**From:** Eddie
**Date:** 2026-02-10

## Request

Deploy the market data infrastructure so portfolio briefings have accurate prices.

## What Exists (in workspace)

1. **`scripts/market_data.py`** — Full portfolio fetcher
   - yfinance for stocks (free, no API key)
   - CoinGecko for crypto (free, rate limited)
   - Commands: `snapshot`, `premarket`, `postclose`, `position TICKER`
   - All 25 positions hardcoded with shares/cost basis

2. **`scripts/portfolio_db.py`** — Database writer for positions
   - Upsert positions with current prices
   - Take daily snapshots
   - Query latest values

## Deployment Steps

### 1. Install yfinance on production
```bash
pip install yfinance
```

### 2. Copy scripts to production
```bash
cp /home/openclaw/.openclaw/workspace/scripts/market_data.py /home/openclaw/homebrew/
cp /home/openclaw/.openclaw/workspace/scripts/portfolio_db.py /home/openclaw/homebrew/
chmod +x /home/openclaw/homebrew/market_data.py
chmod +x /home/openclaw/homebrew/portfolio_db.py
```

### 3. Test
```bash
python3 /home/openclaw/homebrew/market_data.py snapshot
python3 /home/openclaw/homebrew/market_data.py position SSO
```

### 4. Verify cron jobs use the script
The existing 8am/5pm cron prompts should now call:
```bash
python3 /home/openclaw/homebrew/market_data.py premarket   # 8am
python3 /home/openclaw/homebrew/market_data.py postclose   # 5pm
```

## New Table: portfolio_briefings (for embeddings)

Create a table to store daily briefings so we can semantic search them later:

```sql
CREATE TABLE IF NOT EXISTS portfolio_briefings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    briefing_type VARCHAR(20) NOT NULL,  -- 'premarket' or 'postclose'
    briefing_date DATE NOT NULL,
    content TEXT NOT NULL,
    portfolio_value DECIMAL(12,2),
    day_pnl DECIMAL(12,2),
    day_pnl_pct DECIMAL(5,2),
    embedding vector(1536),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, briefing_type, briefing_date)
);

CREATE INDEX idx_portfolio_briefings_embedding ON portfolio_briefings 
USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

CREATE INDEX idx_portfolio_briefings_date ON portfolio_briefings(briefing_date DESC);
```

Then add to db_write.py:
- `store_briefing "premarket|postclose" "content"` — Store with embedding
- `search_briefings "query"` — Semantic search over past briefings

## Why This Matters

Current cron briefings are guessing at prices. With this deployed:
- Accurate real-time prices for all 25 positions
- Daily P&L calculated correctly
- Briefings stored for later analysis
- Can connect journal sentiment to portfolio performance

## Verification

After deployment, Eddie should be able to run:
```bash
python3 /home/openclaw/homebrew/market_data.py snapshot
```
And get JSON with all positions, prices, and daily changes.
