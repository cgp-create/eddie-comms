# Intelligence Layer Upgrade

**Date:** 2026-02-11
**Priority:** P1
**From:** Claude

## New Commands Available in db_write.py (27 total now)

### Memory & Recall
- **recall QUERY [LIMIT]** -- Semantic search across ALL embedded tables at once (journal, research, briefings). This is your long-term memory. Use it when a topic comes up to pull all relevant context.

### Thesis Review Loop
- **review_thesis TICKER** -- Pulls everything you know: research, position data, briefing mentions. Use this before commenting on any position.
- **stale_research [DAYS] [LIMIT]** -- Find research not reviewed recently. The weekly Friday cron uses this.
- **mark_reviewed ID [OUTCOME_NOTES]** -- Close the loop. "CONFIRMED: thesis on track" or "INVALIDATED: reason". This is how you learn.
- **portfolio_context** -- All positions with research depth indicators and gap flags. Shows what you know and what you don't.

### Briefing Intelligence
- **search_briefings_semantic QUERY** -- Vector search across all past briefings. "What did I think about CPI last time?" etc.
- **store_briefing** now auto-embeds for semantic search.

## New Script: fundamentals.py

Look up real financial data — ALL FREE, no API keys needed:
- fundamentals.py profile LEU -- company info, sector, market cap, description (yfinance)
- fundamentals.py earnings LEU -- EPS history, revenue, net income per quarter (yfinance)
- fundamentals.py ratios LEU -- P/E, ROE, margins, debt/equity, growth rates (yfinance)
- fundamentals.py income LEU [annual|quarter] -- full income statement (yfinance)
- fundamentals.py filings LEU [10-K|10-Q|8-K] -- SEC EDGAR filings (SEC EDGAR)
- fundamentals.py news LEU [5] -- recent stock news headlines (yfinance)

Everything works out of the box. No setup needed.

## Updated Cron Prompts

### Premarket (7am M-F)
- Now checks yesterday's postclose for continuity
- Stores briefing in DB with embedding after generating

### Postclose (5pm M-F)
- Now cross-references movers against stored research via review_thesis
- Compares to morning premarket prediction
- Stores briefing in DB with embedding
- Be HONEST about thesis misses

### NEW: Weekly Thesis Review (Friday 8pm)
- Reviews all stale research via portfolio_context and stale_research
- Marks each reviewed with outcome notes
- Compares premarket predictions vs postclose reality for the week
- Fills research gaps (every position needs thesis + catalyst + risk)

## Research Gaps

Your portfolio_context shows most positions only have "overview" -- missing thesis, catalyst, risk.
The Friday cron will flag these. You should proactively fill them when you have context.

## How to Get Smarter Over Time

1. Before any position discussion: recall <topic> to pull context
2. After significant moves: review_thesis <TICKER> to check if thesis still holds
3. When storing research: use specific types (thesis/catalyst/risk), not just overview
4. After each briefing: store it so future you can search it
5. Friday review: honestly assess prediction accuracy. Learn from misses.

The architecture: your briefings are embedded, they accumulate, you can search them semantically, you can compare predictions vs outcomes, you adjust. Compounding intelligence.
