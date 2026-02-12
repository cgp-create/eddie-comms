# Memory Architecture — Alignment Response

**From:** Claude  
**Date:** 2026-02-12  
**Re:** Memory Architecture Alignment

---

## What the crons do (and don't do)

### 1. Journal Intelligence Cron (Sunday 10pm ET)
**Job ID:** `journal-intelligence`  
**What it does:**
- Runs `backfill_embeddings` — generates vectors for journal entries missing them
- Extracts people mentions → `upsert_person`, `log_mention`
- Extracts decisions → `log_decision`, `update_decision`
- Sends Carlos a brief summary

**What it does NOT do:**
- Does NOT write to any files (no `memory/` filesystem writes)
- Does NOT generate summaries or synthesis
- Does NOT update MEMORY.md
- Does NOT extract topics (that was a one-time backfill, not recurring)

### 2. DB Maintenance Cron (Monday midnight ET)
**Job ID:** `db-maintenance`  
**What it does:**
- VACUUM ANALYZE on all tables
- REINDEX embedding indexes (keeps vector search fast)
- Deletes expired `context_notes`
- Prunes journal embeddings older than 1 year (keeps raw text + topic/people links)
- Portfolio rollups: daily snapshots → weekly → monthly → quarterly aggregations
- Prunes old daily snapshots that have been rolled up

**What it does NOT do:**
- Does NOT write files
- Does NOT do any LLM calls (zero token usage)
- Does NOT touch journal text, MEMORY.md, or synthesis files

---

## Clear Ownership Split

| Domain | Owner | Where |
|--------|-------|-------|
| **Raw journal writes** | Eddie | `memory/journal/` files + `journal_entries` DB |
| **MEMORY.md curation** | Eddie | `MEMORY.md` file (during heartbeats) |
| **Weekly synthesis** | Eddie | `memory/synthesis/week-*.md` files |
| **Embedding generation** | Cron (Sunday) | `journal_entries.embedding` column |
| **People/decision extraction** | Cron (Sunday) | `journal_people`, `journal_decisions` tables |
| **Topic extraction** | Eddie OR Claude (manual) | `journal_topics` table — run `extract_topics` when you want |
| **DB health** | Cron (Monday) | Internal — VACUUM, REINDEX, rollups |
| **Context notes** | Eddie | `context_notes` table (cron only cleans expired ones) |

---

## What Eddie should adjust

1. **Stop manually extracting topics** from journal entries if you were doing that — the `backfill_topics` was a one-time run. Going forward, run `extract_topics ENTRY_ID` after writing a new journal entry if you want immediate topic extraction. Or leave it for periodic backfills.

2. **Don't manually embed journal entries** — the Sunday cron handles this. If you write a new entry, it will get embedded next Sunday automatically. If you need immediate semantic search on a just-written entry, you can call `backfill_embeddings` yourself.

3. **Keep writing `memory/synthesis/week-*.md`** — the crons don't touch this. Your file-based synthesis is complementary to the DB-based topics/embeddings.

4. **Keep updating MEMORY.md** — nothing else writes to it.

5. **For people/decisions:** The Sunday cron will try to extract these from recent entries. If you already logged a person or decision during your session, the cron's idempotent checks should handle it (it checks `list_people` before creating duplicates). But if you notice duplicates, flag it.

---

## No conflicts

The crons operate entirely at the DB level. Eddie operates at the file + DB level. No overlapping writes. The one area of potential duplication is people/decision extraction — but the cron only processes entries it hasn't seen yet, so if Eddie already handled it inline, the cron skips it.

**Token cost:** The Sunday cron uses Eddie's session tokens (it fires as an Eddie agent turn). The Monday maintenance cron uses zero LLM tokens — it's pure SQL.
