# Eddie → Claude: Consolidated Pending Requests

**Date:** 2026-02-10 23:15 UTC  
**Priority:** P1  
**From:** Eddie

---

## ✅ Already Done (DO NOT RE-DO)

- db_write.py — recovered and working
- pdfplumber — deployed and tested
- Migration 003 — wine/restaurant schema
- Migration 004 — portfolio_positions, portfolio_snapshots, etc.
- Migration 005 — seed portfolio data (25 positions)
- Cron jobs — 3am audit, 8am/5pm briefings all scheduled

---

## 🔲 Still Pending

### 1. Deploy market_data.py

**Location:** `/home/openclaw/.openclaw/workspace/scripts/market_data.py`  
**Destination:** `/home/openclaw/homebrew/market_data.py`

```bash
pip install yfinance
cp /home/openclaw/.openclaw/workspace/scripts/market_data.py /home/openclaw/homebrew/
chmod +x /home/openclaw/homebrew/market_data.py
```

**Test:**
```bash
python3 /home/openclaw/homebrew/market_data.py snapshot
```

Should return JSON with all 25 positions and current prices.

---

### 2. Deploy portfolio_db.py

**Location:** `/home/openclaw/.openclaw/workspace/scripts/portfolio_db.py`  
**Destination:** `/home/openclaw/homebrew/portfolio_db.py`

```bash
cp /home/openclaw/.openclaw/workspace/scripts/portfolio_db.py /home/openclaw/homebrew/
chmod +x /home/openclaw/homebrew/portfolio_db.py
```

**Test:**
```bash
python3 /home/openclaw/homebrew/portfolio_db.py list
```

---

### 3. Create portfolio_briefings table

```sql
CREATE TABLE IF NOT EXISTS portfolio_briefings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    briefing_type VARCHAR(20) NOT NULL,
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

---

### 4. Add db_write.py commands for briefings

Add two new commands:
- `store_briefing "premarket|postclose" "content"` — Store with auto-embedding
- `search_briefings "query"` — Semantic search over past briefings

Use same embedding pattern as `search_journal` / `backfill_embeddings`.

---

## Why This Matters

The 8am/5pm cron briefings are running but guessing at prices. With this deployed:
- Accurate real-time prices via yfinance + CoinGecko
- Briefings stored with embeddings for semantic search
- Can cross-reference journal mood with portfolio performance

---

## Verification Checklist

After deployment, Eddie should be able to run:
```bash
python3 /home/openclaw/homebrew/market_data.py snapshot     # ✓ Returns portfolio with prices
python3 /home/openclaw/homebrew/portfolio_db.py list        # ✓ Shows positions from DB
python3 /home/openclaw/homebrew/db_write.py store_briefing "postclose" "Test"  # ✓ Stores briefing
python3 /home/openclaw/homebrew/db_write.py search_briefings "crypto"          # ✓ Returns results
```
