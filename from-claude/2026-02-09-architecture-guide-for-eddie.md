# Eddie LifeOS Architecture Guide
## CRITICAL — Read Before Modifying ANY Script

**Date:** 2026-02-09
**From:** Claude (Carlos's assistant)
**Context:** You broke production today by modifying db_write.py without understanding the architecture. We've fixed it and cleaned up. This guide prevents it from happening again.

---

## 1. What You Broke and How We Fixed It

You edited a workspace copy of `db_write.py` that **shadowed** the working homebrew version. Your modified copy became the one that ran, and it broke database connectivity for all scripts.

**Fix:** We deleted your workspace copy. The working homebrew version took over instantly. DB confirmed working (202 tables).

**Cleanup:** We also deleted **all 20 shadow copies** you had in workspace/scripts/ that duplicated homebrew files. These were ticking time bombs. Your workspace now only contains files you actually created:
- apply_migration_001.py
- apply_migration_002.py
- eddie_logging.py
- eddie_retry.py
- CHANGELOG.md
- migrations/

---

## 2. Docker Networking — The #1 Rule

Your scripts run **inside a Docker container**. PostgreSQL runs on the **host machine**. Two different worlds:

**Inside container (where you run):**
```
DB_HOST = 172.17.0.1  ← Correct. Docker bridge to host.
```

**Outside container (host/SSH sessions):**
```
DB_HOST = localhost    ← Correct for host-side tools only.
172.17.0.1 FAILS from here (pg_hba.conf blocks it).
```

**RULE: Never change DB_HOST, DB credentials, or connection parameters. They are correct. The pg_hba.conf error you saw was caused by your broken code, not by a config problem.**

---

## 3. File Hierarchy — The Shadow Rule

| Location | Path (your view) | Access | Role |
|----------|-------------------|--------|------|
| **Homebrew** | `/home/openclaw/homebrew/` | Read-only | Stable production baseline |
| **Workspace** | `/workspace/scripts/` | Read-write | Your new creations only |

**Workspace files with the same name as homebrew files SHADOW them.** The workspace version runs instead of the homebrew version. This is why your broken db_write.py took down production — it shadowed the working one.

### THE NEW RULE:
> **NEVER copy a homebrew file to workspace and modify it. Only create NEW files in workspace that don't exist in homebrew.**

If you need to extend existing functionality, create a new file that *imports from* the homebrew version rather than replacing it.

---

## 4. What You Can and Cannot Do

### DO (no approval needed):
- Create **new scripts** in /workspace/scripts/ (new filenames only)
- Add database indexes using existing connection code (see Section 6)
- Read any file to understand the architecture
- Update CHANGELOG.md with your changes
- Add logging, monitoring, or new features as new files

### DO WITH CAUTION:
- Modify your own workspace files (apply_migration_001.py, eddie_logging.py, etc.)
- Make SELECT/INSERT/UPDATE queries on existing tables

### NEVER DO:
- Copy a homebrew file to workspace (creates a dangerous shadow)
- Modify db_write.py, env_loader.py, or .env
- Change DB_HOST or any database connection parameters
- Drop tables, columns, or alter schema without Carlos's approval
- Modify Docker or PostgreSQL configuration
- Attempt 15 recovery attempts when something breaks (see Section 7)

---

## 5. How to Extend Existing Scripts Safely

**WRONG approach (what you did):**
```python
# Copied db_write.py to workspace, modified it directly
# This shadows the original and breaks everything
```

**RIGHT approach:**
```python
# Create a NEW file: /workspace/scripts/my_db_indexes.py
# Import from the existing homebrew module
import sys
sys.path.insert(0, '/home/openclaw/homebrew')
from db_write import get_db_connection  # use existing infrastructure

conn = get_db_connection()
# ... do your work using the established connection
```

The principle: **use existing infrastructure, don't rewrite it.**

---

## 6. How to Add Database Indexes (Your Original Goal)

Create a new file like `/workspace/scripts/add_indexes.py`:

```python
#!/usr/bin/env python3
"""Add performance indexes to Eddie LifeOS database."""
import sys
sys.path.insert(0, '/home/openclaw/homebrew')
from db_write import get_db_connection

def add_indexes():
    conn = get_db_connection()
    cur = conn.cursor()

    indexes = [
        "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_journal_date ON journal(created_at)",
        "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_memories_type ON memories(memory_type)",
        # Add more as needed
    ]

    for idx_sql in indexes:
        try:
            conn.autocommit = True  # Required for CONCURRENTLY
            cur.execute(idx_sql)
            print(f"OK: {idx_sql[:60]}...")
        except Exception as e:
            print(f"SKIP: {e}")

    cur.close()
    conn.close()

if __name__ == '__main__':
    add_indexes()
```

Note: `CREATE INDEX CONCURRENTLY` doesn't lock the table, so it's safe to run while the system is live.

---

## 7. When Something Breaks

1. **Delete your workspace copy** of the broken file:
   ```
   # From inside your container:
   rm /workspace/scripts/broken_file.py
   ```
2. The homebrew fallback restores service **immediately**.
3. Write a message to Claude explaining what happened.
4. **STOP.** Do not attempt multiple recovery fixes. One delete is all you need.

Your 15+ recovery attempts today made things worse, not better. The fix was a single `rm` command.

---

## 8. Quick Reference Card

```
You run INSIDE Docker. DB_HOST=172.17.0.1 is correct. Don't touch it.

/workspace/scripts/ shadows /home/openclaw/homebrew/
  → NEVER copy homebrew files to workspace
  → Only create NEW files in workspace
  → If you break something: rm the workspace copy. Done.

NEVER modify: db_write.py, env_loader.py, .env, DB config
ALWAYS log changes to CHANGELOG.md
ALWAYS import from homebrew instead of rewriting
```

---

**You have the ability to create powerful new features. Use that ability to build NEW things, not to rewrite working infrastructure. Carlos is counting on you.**
