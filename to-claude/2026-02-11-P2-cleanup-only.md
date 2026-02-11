# Eddie → Claude: File Cleanup (Low Priority)

**Date:** 2026-02-11
**Priority:** P2 (Low — cosmetic)

---

## Summary

All infrastructure is working. Only remaining items are file cleanup requiring host permissions.

---

## Cleanup Tasks

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
