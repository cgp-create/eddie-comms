# COMPREHENSIVE FIX: Media File Permissions

**Date:** 2026-02-09 14:11 UTC  
**Priority:** P1 (High - blocks PDF ingestion workflow)  
**Status:** Root cause analysis complete, requesting permanent fix

**NOTE:** Supersedes `2026-02-09-permission-issue.md` and `2026-02-09-file-permissions-ongoing.md` (delete those - they were incremental thinking, not complete analysis)

---

## Problem Statement

PDFs arriving in `/workspace/media/inbound/` are created with mode 600 (owner-only read/write), preventing the sandboxed agent container from reading them.

---

## Evidence

```bash
ls -la /workspace/media/inbound/*.pdf | tail -3
-rw-r--r-- 1 1000 1000 40324 Feb  9 13:45 file_28...pdf  # ✅ Manually fixed by you
-rw------- 1 1000 1000 25364 Feb  9 14:04 file_29...pdf  # ❌ Still restricted

# Container runs as root but can't access uid 1000 mode 600 files
id
→ uid=0(root)

python3 /home/openclaw/homebrew/pdf_extract.py extract /workspace/media/inbound/file_29...pdf
→ PermissionError: [Errno 13] Permission denied
```

---

## Root Cause

Files are being created by the OpenClaw media handler (gateway process) with restrictive umask or explicit mode 600.

**Why this matters:**
- Gateway likely runs as uid 1000 on host
- Creates files with mode 600 (owner-only)
- Container runs as root (uid 0) but volume-mounted files retain host permissions
- Container root ≠ host uid 1000, so no access despite being "root"

---

## Attempted Band-Aids (Don't Do This)

❌ Manual `chmod 644` on each file (doesn't scale)  
❌ inotify watcher to auto-chmod (reactive, not root cause fix)  
❌ Copy files to different location (unnecessary complexity)

---

## Correct Fix: Change File Creation Permissions

**Location:** OpenClaw gateway media handler (wherever files are written to `media/inbound/`)

**Change needed:**

```python
# WRONG (current behavior)
with open(filepath, 'wb') as f:
    f.write(data)
# Creates file with umask 077 → mode 600

# RIGHT (what we need)
with open(filepath, 'wb') as f:
    f.write(data)
os.chmod(filepath, 0o644)  # Explicitly set readable by all

# OR set umask before file creation
old_umask = os.umask(0o022)  # umask 022 → mode 644 for files
with open(filepath, 'wb') as f:
    f.write(data)
os.umask(old_umask)
```

**Target permissions:** `-rw-r--r--` (mode 644)
- Owner (uid 1000): read+write
- Group: read
- Others (including container root): read

---

## Alternative Fix: Run Container as uid 1000

**Change:** Run agent container as uid 1000 instead of root

**Pros:**
- Container would own the files (same uid)
- No permission issues
- Better security (least privilege)

**Cons:**
- Might break other things expecting root
- Needs broader testing

**If you choose this path:**
```dockerfile
USER 1000:1000  # in Dockerfile
```

---

## Testing

Once fixed, verify:

```bash
# New files should arrive with mode 644
ls -la /workspace/media/inbound/
# Should see: -rw-r--r-- 1 1000 1000 ...

# Container should read without error
python3 /home/openclaw/homebrew/pdf_extract.py extract /workspace/media/inbound/file_29...pdf
# Should return: {"status": "success", ...}
```

---

## Impact

**Current:** Manual intervention required for every PDF Carlos sends  
**Post-fix:** PDFs auto-readable immediately after arrival

**Workflow dependency:** Carlos is sending knowledge base PDFs for ingestion. This blocks that entire workflow until fixed.

---

## Recommendation

**Primary fix:** Change media handler to create files with mode 644 (or umask 022)  
**Fallback:** Run container as uid 1000 if that's architecturally simpler

**Test files waiting:**
- `file_29---220edd9d-95ee-4f28-8d92-a92172232b50.pdf` (25KB, FirstMark interview prep)
- Any future PDFs Carlos sends

Please implement the permanent fix at file creation time.
