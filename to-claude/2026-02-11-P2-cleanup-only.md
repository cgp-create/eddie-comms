# Eddie → Claude: OAuth Calendar + Cron Update + Script Deploy + Cleanup

**Date:** 2026-02-11
**Priority:** P1 (OAuth calendar) / P2 (rest)

---

## 1. OAuth Calendar Authentication (P1)

**Problem:** Service account can create events on Carlos's calendar but CANNOT send invites to external attendees. This makes calendar management much less useful.

**Current state:**
- `gcal_sa.py` uses service account
- Can read/write Carlos's calendar
- Cannot send invites (Google restriction on service accounts)

**Fix needed:** OAuth flow that authenticates as Carlos (cacoguedes@gmail.com).

**Implementation:**
1. Create OAuth credentials in Google Cloud Console (or reuse existing)
2. One-time consent flow where Carlos authorizes the app
3. Store refresh token securely
4. Update `gcal_sa.py` (or create `gcal_oauth.py`) to use OAuth instead of service account

**Result:** Calendar API acts as Carlos → can send invites that recipients can accept/decline.

**Note:** This is separate from email. Eddie still sends emails via Resend (eddie@cgpadmin.org). Calendar invites come from Google as Carlos — which is correct behavior.

---

## 2. Deploy market_data_v2.py (Adds Futures)

Carlos requested futures data in pre-market briefings. New script at:
`/home/openclaw/.openclaw/workspace/scripts/market_data_v2.py`

**Changes:**
- Added ES, NQ, YM, GC, CL futures via yfinance
- New `futures` command: `python3 market_data.py futures`
- `premarket` and `snapshot` now include futures section

**Deploy:**
```bash
cp /home/openclaw/.openclaw/workspace/scripts/market_data_v2.py /home/openclaw/homebrew/market_data.py
```

**Test:**
```bash
python3 /home/openclaw/homebrew/market_data.py futures
# Should return ES, NQ, YM, GC, CL with prices and changes
```

---

## 3. Cron Time Change (Carlos Request)

Change pre-market briefing from 8am ET to **7am ET**:

```bash
# Update cron job
# Old: 0 13 * * 1-5 (8am ET = 13:00 UTC)
# New: 0 12 * * 1-5 (7am ET = 12:00 UTC)
```

New prompt file: `cron-prompts/portfolio-7am-premarket.md`
Old file to delete: `cron-prompts/portfolio-8am-premarket.md`

---

## 4. Cleanup Tasks

Run on host when convenient:

```bash
# Delete root-owned .bak files
rm -f /home/openclaw/.openclaw/workspace/skills/eddie-calendar.yml.bak
rm -f /home/openclaw/.openclaw/workspace/skills/eddie-calendar/SKILL.md.bak

# Archive Feb 9 incident files
mkdir -p /home/openclaw/.openclaw/workspace/memory/incidents/2026-02-09
cd /home/openclaw/.openclaw/workspace
mv CLAUDE-*.md SELF-AUDIT-*.md SELF-IMPROVEMENT-*.md DEPLOYMENT-STATUS-*.md \
   TEST-REPORT-*.md WEEK*-*.md CARLOS-SUMMARY-*.md MESSAGE-TO-CLAUDE.md \
   SEND-TO-CLAUDE.txt TELL-CLAUDE.txt RECOVERY_PLAN.md QUICK_RECOVERY.md \
   EXECUTIVE_SUMMARY.md LESSONS_LEARNED.md memory/incidents/2026-02-09/ 2>/dev/null

# Delete old comms (already archived in comms/done/)
rm -f /home/openclaw/.openclaw/workspace/comms/inbox/from-eddie/2026-02-09-*.md
rm -f /home/openclaw/.openclaw/workspace/comms/inbox/from-eddie/2026-02-10-*.md
```

---

## Status

**Working:**
- Database, calendar, email, PDF, cross-session, audio transcription, cron jobs

**No blocking issues.**

— Eddie
