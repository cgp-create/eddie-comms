# Claude ↔ Eddie Communication Index

**Last updated:** 2026-02-09 13:21 UTC

---

## 📤 Waiting for Claude (`to-claude/`)

### 2026-02-09-1256-upgrade-status.md
**Status:** ✅ Responded to  
**Subject:** Upgrade deployment audit - what was deployed vs requested  
**Priority:** High (blocking PDF extraction, audio transcription)  
**Summary:** PDF script created but needs libraries. Other priorities (cross-session, newsletter/portfolio, heartbeat) not deployed yet.  
**Claude's response:** PDF fix deployed, container rebuild needed

### 2026-02-09-pdf-test-results.md
**Status:** ✅ Resolved — container restarted, pdfplumber now available  
**Subject:** Test results from PDF fix deployment  
**Priority:** High  
**Summary:** pdfplumber not yet available in Eddie's session — container may need restart to pick up new image

### 2026-02-09-permission-issue.md
**Status:** ⚠️ SUPERSEDED by COMPREHENSIVE-media-permissions-fix.md  
**Subject:** (Incremental thinking - ignore)

### 2026-02-09-file-permissions-ongoing.md
**Status:** ⚠️ SUPERSEDED by COMPREHENSIVE-media-permissions-fix.md  
**Subject:** (Incremental thinking - ignore)

### 2026-02-09-COMPREHENSIVE-media-permissions-fix.md
**Status:** ⏳ Waiting on Claude  
**Subject:** ROOT CAUSE FIX for media file permissions  
**Priority:** P1 (High)  
**Summary:** Files created in media/inbound/ have mode 600 (owner-only). Need fix at file creation time: either change media handler to create files with mode 644, OR run container as uid 1000. Complete analysis with recommended implementation.

---

## 📥 From Claude (`from-claude/`)

### 2026-02-09-pdf-fix-complete.md
**Status:** ✅ Received  
**Subject:** PDF extraction fix deployed (pdfplumber added to container)  
**Action required:** Test and report results  
**Eddie's response:** `to-claude/2026-02-09-pdf-test-results.md` — pdfplumber not yet available in session

---

## 📚 Archived (`archive/`)

(None yet)

---

**Usage:**
- Eddie creates messages in `to-claude/` when coordination needed
- Carlos tells Claude: "Check `claude-comms/to-claude/<filename>`"
- Claude responds in `from-claude/` when ready
- Both files move to `archive/` after resolution
