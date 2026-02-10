# PRIORITY 0: Database Authentication Crisis

**Date:** 2026-02-09 21:00 UTC  
**Status:** PRODUCTION DOWN - Multiple scripts failing  
**Severity:** CRITICAL

## What Happened

Carlos asked Eddie to add database performance indexes. Eddie broke production.

## Timeline

1. **20:24 UTC** - Carlos asks about database indexes and 3-tier model routing
2. **20:30-20:55 UTC** - Eddie attempts to add indexes, breaks `/home/openclaw/homebrew/db_write.py`
3. **20:55 UTC** - Sub-agent attempts recovery, creates "fixed" version
4. **21:00 UTC** - ALL database scripts now failing with pg_hba.conf errors

## Current Error

```
psycopg2.OperationalError: connection to server at "172.17.0.1", port 5432 failed: FATAL: no pg_hba.conf entry for host "138.197.89.76", user "eddie", database "eddie_lifeos", SSL encryption
connection to server at "172.17.0.1", port 5432 failed: FATAL: no pg_hba.conf entry for host "138.197.89.76", user "eddie", database "eddie_lifeos", no encryption
```

## Affected Scripts

- ❌ `/home/openclaw/homebrew/db_write.py` - BROKEN (Eddie overwrote it)
- ❌ `/home/openclaw/homebrew/eddie_memory.py` - NOW FAILING (was working earlier)
- ❓ Other DB-dependent scripts unknown

## What Eddie Did Wrong

1. **Edited sandbox copy** (`/home/openclaw/.openclaw/workspace/scripts/db_write.py`) instead of production
2. **Copied sandbox to production** without understanding architectural differences
3. **Broke working file** that wasn't tracked in git
4. **Made 15+ recovery attempts** without success
5. **Possibly triggered database authentication changes** (unclear how)

## Questions for You

1. **Why are scripts suddenly unable to connect to PostgreSQL?**
   - Earlier today, scripts like `eddie_memory.py` worked fine
   - Now getting pg_hba.conf rejections from IP 138.197.89.76
   - Is this IP the droplet's public IP trying to connect to Docker PostgreSQL?

2. **What's the correct architecture?**
   - Where should database scripts run? (droplet vs container)
   - What's the proper connection path?
   - Why does DB_HOST=172.17.0.1 in .env?

3. **How to recover db_write.py?**
   - File not tracked in git
   - No backups found
   - Can you reconstruct from working patterns?

4. **Did Eddie break PostgreSQL configuration?**
   - Or is this a pre-existing auth issue that coincidentally surfaced?

## Environment Info

**Database credentials** (from `/home/openclaw/homebrew/.env`):
```
DB_HOST=172.17.0.1
DB_NAME=eddie_lifeos
DB_USER=eddie
DB_PASS=eddie_lifeos_2026
```

**Droplet:** CGP-LifeOS
**PostgreSQL:** Running in Docker (172.17.0.1)
**Scripts location:** `/home/openclaw/homebrew/`

## What Carlos Needs

1. **Working db_write.py** restored (recent_journal, recent_workouts, apply_indexes commands)
2. **Database connectivity** fixed for all scripts
3. **Architectural documentation** so Eddie doesn't repeat this
4. **Database indexes** added (original goal, now secondary)

## Immediate Risk

Carlos is losing trust in Eddie's ability to:
- Understand architectural boundaries
- Make safe changes without breaking production
- Recover from mistakes

## Request

Please provide:
1. Root cause analysis of pg_hba.conf errors
2. Recovery steps for database connectivity
3. Reconstructed db_write.py (or recovery method)
4. Architectural guide (sandbox vs production, docker networking)
5. Guardrails to prevent Eddie from breaking production again

Carlos and Eddie are standing by for your guidance.

---

**Attachments:**
- Error screenshots available if needed
- Can provide any log files or configurations you need
