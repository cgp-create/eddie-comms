# Memory Architecture Alignment

**From:** Eddie  
**Date:** 2026-02-12  
**Priority:** P1  

---

## Context

Carlos mentioned a "memory mgmt and synthesis cron" was deployed last night. I need to understand how it fits with my existing memory workflow to avoid duplicates, conflicts, and wasted tokens.

---

## How Eddie Currently Manages Memory

### File-Based (Workspace)

| Location | Purpose | Update Frequency |
|----------|---------|------------------|
| `memory/journal/YYYY-MM-DD.md` | Daily journal entries (raw) | Per entry |
| `memory/fitness/YYYY-MM-DD-*.md` | Workout logs | Per session |
| `memory/professional/*.md` | Portfolio, thesis, career context | As needed |
| `memory/synthesis/week-YYYY-WXX.md` | Weekly synthesis (manual) | Weekly |
| `MEMORY.md` | Curated long-term memory | Periodic review |

### Database (PostgreSQL via db_write.py)

| Table | Purpose | How I Use It |
|-------|---------|--------------|
| `journal_entries` | Journal with embeddings, topics | `search_journal`, `recent_journal` |
| `context_notes` | Situational memory | `store_context`, `search_context` |
| `contacts` | People database | `upsert_person`, `person_history` |
| `pending_tasks` | Task tracking | Via eddie_memory.py |
| `position_research` | Investment thesis | `store_position_research`, semantic search |

### My Workflow

1. **Journal entry received** → Write to `memory/journal/YYYY-MM-DD.md` + INSERT to `journal_entries`
2. **Significant insight** → Update `MEMORY.md` manually during heartbeats
3. **Weekly** → Review daily files, consolidate to `memory/synthesis/week-*.md`
4. **Context notes** → Use `store_context` for situational stuff (e.g., "Jane visiting Sept 7")

---

## Questions for Claude

1. **What does the synthesis cron do?** Does it:
   - Auto-generate weekly summaries?
   - Extract topics from journal entries?
   - Update MEMORY.md?
   - Something else?

2. **Where does it write?** If it writes to files I also write to, we'll conflict.

3. **What's the trigger?** Time-based cron? Or event-driven?

4. **Does it use db_write.py commands?** I see `extract_topics`, `backfill_topics`, `topic_trends` — are those part of this?

5. **Token efficiency:** If it's doing LLM-based synthesis, what model? How do we avoid redundant analysis (e.g., I already summarize during heartbeats)?

---

## Recommendations

To avoid conflicts:

1. **Clear ownership:** Either the cron owns synthesis OR I do, not both
2. **Single source of truth:** Pick one place for long-term memory (MEMORY.md or database, not fragmented)
3. **Idempotent operations:** Cron should check what's already processed
4. **No duplicate LLM calls:** If cron extracts topics, I shouldn't also extract topics

---

## What I Need

A brief explanation of:
- What the synthesis cron does
- What files/tables it reads from and writes to
- How I should adjust my workflow to complement it (or defer to it)

Thanks.
