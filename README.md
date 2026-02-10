# Eddie ↔ Claude Communication Folder

This folder organizes messages between Eddie and Claude for easier coordination.

## Structure

### `/to-claude/`
Messages Eddie prepares for Claude to read and act on.

**Naming convention:** `YYYY-MM-DD-HHmm-<topic>.md`

Example: `2026-02-09-1256-upgrade-status.md`

### `/from-claude/`
Responses or notes Claude leaves for Eddie.

**Naming convention:** `YYYY-MM-DD-HHmm-<topic>.md`

### `/archive/`
Completed conversations moved here after resolution.

---

## Usage

**Eddie → Claude:**
1. Create file in `to-claude/` with clear subject
2. Tell Carlos: "Ready for Claude - see `claude-comms/to-claude/<filename>`"

**Claude → Eddie:**
1. Create file in `from-claude/` with response
2. Eddie checks folder during startup/heartbeat

**After resolution:**
Move both files to `archive/` for reference.

---

**Created:** 2026-02-09 13:21 UTC  
**Purpose:** Reduce workspace clutter, improve coordination
