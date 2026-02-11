# P1: Infrastructure Build - PDF, Sessions, Newsletter, Portfolio

**Priority:** 1 (High - blocking workflow capabilities)  
**Status:** Needs implementation  
**From:** Eddie  
**Date:** 2026-02-09

---

## Overview

Eddie's database foundation is complete. These are the remaining infrastructure gaps blocking full functionality.

---

## 🔴 PRIORITY 0: PDF EXTRACTION (BLOCKING)

**Problem:** Eddie can't read PDFs. No PyPDF2, pdfplumber, or pdftotext available.

**Evidence:** Carlos sent investment docs 3 times - all failed.

**Test file waiting:** `media/inbound/file_23---c149794c-0f2e-41a3-8564-37b06761991e.pdf` (40KB, 16 pages)

**Fix needed:**
```bash
# Install PDF library
apt install python3-pypdf2

# Create extraction script
cat > /home/openclaw/homebrew/pdf_extract.py << 'EOF'
#!/usr/bin/env python3
import sys
try:
    import PyPDF2
    with open(sys.argv[1], 'rb') as f:
        reader = PyPDF2.PdfReader(f)
        for page in reader.pages:
            print(page.extract_text())
except Exception as e:
    print(f"Error: {e}", file=sys.stderr)
    sys.exit(1)
EOF
chmod +x /home/openclaw/homebrew/pdf_extract.py
```

**Test:**
```bash
python3 /home/openclaw/homebrew/pdf_extract.py media/inbound/file_23---c149794c-0f2e-41a3-8564-37b06761991e.pdf
# Should output: readable investment thesis text
```

---

## 🔴 PRIORITY 1: CROSS-SESSION MESSAGING (GROUP CHAT)

**Problem:** Eddie can't send messages to other sessions. Sandbox blocks cross-session access.

**Evidence:**
- `sessions_list()` returns count: 0 (no sessions visible)
- `sessions_send()` fails: "Session not visible from this sandboxed agent session"
- Carlos confirmed Eddie has replied in group "CGP x MJQ x Eddie" before

**Test case that fails:**
1. Carlos sends instruction in 1:1: "Tell Melissa something nice in the group chat"
2. Eddie tries: `sessions_send --label "CGP x MJQ x Eddie" --message "..."`
3. Result: "forbidden - Session not visible"

**Fix needed:**
- Investigate OpenClaw sandbox session visibility settings
- Enable cross-session access for Eddie's main agent
- Or: propose different architecture for multi-chat handling

**Test after fix:**
```bash
sessions_list  # Should show: group chat session
sessions_send --label "CGP x MJQ x Eddie" --message "Test"
# Should: Message appears in group chat
```

---

## 📊 PRIORITY 2: NEWSLETTER + PORTFOLIO INFRASTRUCTURE

### Database Tables Needed

```sql
-- Newsletter tracking
CREATE TABLE newsletter_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source TEXT NOT NULL,
  date DATE NOT NULL,
  raw_content TEXT,
  themes TEXT[],
  created_at TIMESTAMP DEFAULT NOW()
);

-- Portfolio tracking
CREATE TABLE portfolio (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ticker TEXT NOT NULL UNIQUE,
  asset_class TEXT,
  position_size DECIMAL,
  thesis TEXT,
  entry_date DATE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Daily synthesis
CREATE TABLE daily_synthesis (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  date DATE NOT NULL UNIQUE,
  key_themes TEXT[],
  portfolio_impact TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### Scripts Needed

**1. Newsletter processor:** `/home/openclaw/homebrew/newsletter_process.py`

Commands to support:
```bash
python3 /home/openclaw/homebrew/newsletter_process.py scrape --last-24h
python3 /home/openclaw/homebrew/newsletter_process.py synthesize --date today
python3 /home/openclaw/homebrew/newsletter_process.py sources
```

**Flow:**
- Scrape newsletters from eddie@cgpadmin.org inbox
- Extract text, tag themes, store in newsletter_items
- On demand: Opus synthesizes across newsletters + portfolio
- Output: top trends, portfolio impacts, contradictions, opportunities, risks
- Store in daily_synthesis

**2. Portfolio importer:** `/home/openclaw/homebrew/portfolio_import.py`

Commands to support:
```bash
python3 /home/openclaw/homebrew/portfolio_import.py --file portfolio.pdf
python3 /home/openclaw/homebrew/portfolio_import.py add --ticker NVDA --thesis "..."
python3 /home/openclaw/homebrew/portfolio_import.py list
```

*(Blocked by PDF extraction - fix that first)*

---

## ⚡ PRIORITY 3: HAIKU MODEL INTEGRATION

**Goal:** Use Haiku ($0.25/$1.25 per MTok) for mechanical ops instead of Sonnet ($3/$15).

**What needs Haiku routing:**
- Heartbeat checks (2x daily)
- Newsletter scraping
- Database logging
- Email inbox listing

**Implementation:**
- Add `--model haiku` parameter to scripts
- Default heartbeat to Haiku
- Keep Sonnet for conversation/analysis
- Reserve Opus for deep synthesis

**Cost savings:** ~$0.50/month at current volume, scales with usage.

---

## ⏰ PRIORITY 4: HEARTBEAT AUTOMATION

**Setup cron:**
```bash
# Morning check - 8am UTC (3am EST)
0 8 * * * openclaw heartbeat run morning

# Evening check - 8pm UTC (3pm EST)
0 20 * * * openclaw heartbeat run evening
```

**Morning heartbeat (using Haiku):**
- Check inbox for urgent emails
- Check calendar conflicts today
- Run newsletter scraper if new items
- Check tasks due today
- Return "HEARTBEAT_OK" if nothing urgent, else notify

**Evening heartbeat (using Haiku):**
- Has Carlos journaled today? Nudge if not
- Inbox scan again
- Tomorrow's calendar preview
- Workout logged? (if training day)
- Return "HEARTBEAT_OK" or relevant check-in

---

## 🛠️ PRIORITY 5: SCRIPT ENHANCEMENTS

**Add to `/home/openclaw/homebrew/db_write.py`:**

```bash
# Log champagne
python3 /home/openclaw/homebrew/db_write.py log_champagne --name "Dom Pérignon 2012" --rating 9 --notes "..." --price "$$$"

# Log restaurant
python3 /home/openclaw/homebrew/db_write.py log_restaurant --name "Carbone" --rating 8 --cuisine Italian --notes "..." --status favorite

# List newsletters
python3 /home/openclaw/homebrew/db_write.py list_newsletters --since 2026-02-01

# List portfolio
python3 /home/openclaw/homebrew/db_write.py portfolio_list
```

Optional: Enhance workout logging to accept detailed exercise structure (sets/reps/weight).

---

## ✅ ALREADY WORKING (Confirmed by audit)

Eddie verified these are functional:
- ✅ Database: journal_entries, workouts, users, tasks, contacts (all exist)
- ✅ Structured data: tags and sentiment_score populated in journals
- ✅ Query tools: db_write.py query, recent_journal, recent_workouts all work
- ✅ Email: send/receive functional (eddie@cgpadmin.org)
- ✅ Calendar: gcal_sa.py operational, both calendars accessible
- ✅ Memory system: markdown files, directory structure established

**Don't touch these - they work.**

---

## 🧪 TESTING CHECKLIST

**PDF Extraction:**
```bash
python3 /home/openclaw/homebrew/pdf_extract.py media/inbound/file_23---c149794c-0f2e-41a3-8564-37b06761991e.pdf
# Expected: Readable text output (investment thesis)
```

**Cross-Session:**
```bash
sessions_list
# Expected: Show group chat "CGP x MJQ x Eddie"

sessions_send --label "CGP x MJQ x Eddie" --message "Test"
# Expected: Message appears in group chat
```

**Newsletter:**
```bash
python3 /home/openclaw/homebrew/newsletter_process.py scrape --last-24h
python3 /home/openclaw/homebrew/db_write.py query "SELECT COUNT(*) FROM newsletter_items"
# Expected: Rows inserted
```

**Heartbeat:**
```bash
openclaw heartbeat run morning
# Expected: "HEARTBEAT_OK" or alert text

crontab -l | grep heartbeat
# Expected: 2 cron entries at 8am/8pm UTC
```

**Database:**
```bash
python3 /home/openclaw/homebrew/db_write.py query "SELECT table_name FROM information_schema.tables WHERE table_schema='public' AND table_name IN ('newsletter_items', 'portfolio', 'daily_synthesis')"
# Expected: All 3 new tables exist
```

---

## ❓ QUESTIONS FOR CLAUDE

1. **Session isolation:** What's the security model for cross-session access? How should Eddie access group chats without breaking sandbox?
2. **PDF library:** Prefer `python3-pypdf2` (apt install) or need workaround for pip in externally-managed environment?
3. **Haiku model:** Already configured in OpenClaw or need new setup?
4. **Newsletter sources:** Start with manual list (Milk Road, Not Boring, etc.) or auto-detect from inbox patterns?
5. **Heartbeat mechanism:** OpenClaw native cron or external scheduler?

---

## SUMMARY

**Critical blockers (fix first):**
1. PDF extraction - Can't read investment docs
2. Cross-session messaging - Can't reply in group chat when instructed from 1:1

**Infrastructure to build:**
3. Newsletter + portfolio tables & scripts
4. Haiku model integration (cost optimization)
5. Heartbeat automation (proactive checks)
6. Script enhancements (champagne, restaurant commands)

**Database foundation is solid.** Focus on the two blockers first - they're breaking live functionality.
