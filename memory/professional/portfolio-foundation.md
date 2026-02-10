# Portfolio & Finance Foundation

**Created:** 2026-02-10
**Status:** Design phase

---

## Database Schema Design

### Table: `portfolio_positions`

Tracks individual positions over time.

```sql
CREATE TABLE portfolio_positions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    snapshot_date DATE NOT NULL,
    
    -- Position identification
    symbol VARCHAR(20) NOT NULL,
    position_name VARCHAR(255),
    asset_class VARCHAR(50),  -- 'equity', 'crypto', 'fixed_income', 'alternative', 'cash'
    account VARCHAR(100),      -- e.g., 'Fidelity', 'Coinbase', 'BlockFi'
    
    -- Position data
    quantity DECIMAL(18, 8),
    cost_basis DECIMAL(18, 2),
    market_value DECIMAL(18, 2),
    
    -- Calculated fields (can be computed or stored)
    weight_pct DECIMAL(5, 2),  -- % of total portfolio
    gain_loss DECIMAL(18, 2),
    gain_loss_pct DECIMAL(8, 2),
    
    -- Metadata
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(user_id, snapshot_date, symbol, account)
);

CREATE INDEX idx_positions_user_date ON portfolio_positions(user_id, snapshot_date);
CREATE INDEX idx_positions_symbol ON portfolio_positions(symbol);
```

### Table: `portfolio_snapshots`

Daily portfolio summary for quick querying.

```sql
CREATE TABLE portfolio_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    snapshot_date DATE NOT NULL,
    
    -- Totals
    total_value DECIMAL(18, 2),
    total_cost_basis DECIMAL(18, 2),
    total_gain_loss DECIMAL(18, 2),
    total_gain_loss_pct DECIMAL(8, 2),
    
    -- Allocation breakdown (stored as JSONB for flexibility)
    allocation_by_class JSONB,  -- {"equity": 45.2, "crypto": 30.1, ...}
    allocation_by_account JSONB,
    
    -- Daily change
    daily_change DECIMAL(18, 2),
    daily_change_pct DECIMAL(8, 2),
    
    -- Metadata
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(user_id, snapshot_date)
);

CREATE INDEX idx_snapshots_user_date ON portfolio_snapshots(user_id, snapshot_date);
```

### Table: `market_context`

Macro data for overlay analysis.

```sql
CREATE TABLE market_context (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    snapshot_date DATE NOT NULL,
    snapshot_time TIME,  -- for intraday (8am, 5pm)
    
    -- Major indices
    sp500 DECIMAL(10, 2),
    sp500_change_pct DECIMAL(8, 2),
    nasdaq DECIMAL(10, 2),
    nasdaq_change_pct DECIMAL(8, 2),
    btc_price DECIMAL(18, 2),
    btc_change_pct DECIMAL(8, 2),
    eth_price DECIMAL(18, 2),
    eth_change_pct DECIMAL(8, 2),
    
    -- Macro indicators
    vix DECIMAL(6, 2),
    dxy DECIMAL(8, 2),  -- dollar index
    us_10y_yield DECIMAL(5, 2),
    
    -- Sentiment/context (from news, can be AI-generated summary)
    market_narrative TEXT,
    key_events TEXT[],
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(snapshot_date, snapshot_time)
);
```

---

## Data Sources

### Portfolio Data
- **Primary:** Manual updates or CSV import from brokerage statements
- **Future:** API integrations (Fidelity, Coinbase, etc.)
- **Baseline:** PDF briefing already ingested (~$770K total)

### Market Data (Free APIs)
- **Yahoo Finance** (yfinance Python library) — equities, indices
- **CoinGecko** — crypto prices
- **FRED** — macro indicators (10Y yield, etc.)

### Macro Context
- **PDFs ingested:** Need to identify which 2 PDFs Carlos sent
- **News:** Can scrape headlines or use free news APIs

---

## Cron Job Design

### Schedule
- **8:00 AM ET** — Pre-market briefing
- **5:00 PM ET** — Post-close summary

### 8 AM Prompt (Pre-Market)
```
Portfolio Pre-Market Briefing.

1. Fetch overnight market moves:
   - S&P 500 futures
   - Nasdaq futures
   - BTC/ETH prices
   - Any major overnight news

2. Check positions with significant pre-market moves (>2%)

3. Identify key events today:
   - Earnings for held positions
   - Fed speakers
   - Economic data releases

4. Summary format:
   MARKETS: [futures, crypto overnight]
   PORTFOLIO ALERT: [any position moving >2%]
   TODAY'S EVENTS: [relevant catalysts]
   
Deliver to Telegram. Keep it tight — 5 bullets max.
```

### 5 PM Prompt (Post-Close)
```
Portfolio Post-Close Summary.

1. Fetch closing prices and calculate:
   - Daily portfolio P&L ($ and %)
   - Biggest winners/losers
   - Portfolio value vs. yesterday

2. Market context:
   - S&P, Nasdaq, BTC/ETH daily change
   - VIX level and change
   - Any macro news that moved markets

3. Store snapshot to database

4. Summary format:
   TODAY: +/-$X,XXX (+/-X.X%)
   PORTFOLIO: $XXX,XXX
   MOVERS: [top 2 up, top 2 down]
   MARKET: [1-line context]
   
Deliver to Telegram. Brief — what mattered, what didn't.
```

---

## Implementation Steps

1. **Create database tables** (need Claude to deploy migration)
2. **Seed baseline positions** from PDF briefing
3. **Build market data fetcher** (Python script using yfinance, coingecko)
4. **Create cron jobs** (once we fix the cron system)
5. **Test and iterate on prompts**

---

## Questions for Carlos

1. Which 2 PDFs have the macro overlay? (Need to identify and extract key thesis)
2. What positions are in the ~$770K portfolio? (Need to seed the database)
3. Which accounts to track? (Fidelity, Coinbase, etc.)
4. Any specific positions to watch more closely?
