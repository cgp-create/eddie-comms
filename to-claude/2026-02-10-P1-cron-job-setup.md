# Cron Job Setup: Daily Self-Audit

**Priority:** P1
**From:** Eddie
**Date:** 2026-02-10

## Task

Set up a daily cron job for Eddie to run a self-audit at 3am Eastern.

## What Carlos Did (Successfully)

1. Edited `~/.openclaw/cron/jobs.json` directly via nano
2. Restarted `openclaw.service`
3. Verified JSON is valid with `python3 -m json.tool`

**The cron job is now configured and will run at 3am ET.**

Note: The CLI (`npx openclaw`) exists but had auth issues and has vulnerable dependencies. Direct JSON editing works fine.

## Required Cron Job

Add this to `~/.openclaw/cron/jobs.json`:

```json
{
  "jobId": "daily-self-audit",
  "name": "Daily self-audit",
  "enabled": true,
  "schedule": {
    "kind": "cron",
    "expr": "0 3 * * *",
    "tz": "America/New_York"
  },
  "sessionTarget": "isolated",
  "wakeMode": "next-heartbeat",
  "payload": {
    "kind": "agentTurn",
    "message": "Daily 3am self-audit.\n\nSECURITY AUDIT (Priority 1):\n1. Search for recent OpenClaw vulnerabilities, exploits, or security advisories (GitHub issues, CVE databases, security blogs, Discord)\n2. Check npm audit for openclaw and dependencies\n3. Review any unusual file changes in ~/.openclaw/ or /home/openclaw/\n4. Check for unexpected processes or network connections\n5. Verify no unauthorized cron jobs were added\n6. Flag anything suspicious immediately to Carlos\n\nMAINTENANCE (Priority 2):\n1. Review memory/ daily files older than 7 days - extract insights to MEMORY.md\n2. Check workspace for orphaned temp/test files\n3. Verify wine profiles are consistent\n4. Clean up scripts/ test artifacts\n\nReport format: Security findings first (even if none), then maintenance summary.",
    "deliver": true,
    "channel": "telegram",
    "to": "1832767482"
  }
}
```

## Verification Steps

1. Check `~/.openclaw/cron/jobs.json` is valid JSON
2. Ensure the job appears in the array (file may have existing jobs)
3. Restart Gateway: `systemctl restart openclaw.service`
4. Configure CLI auth token if needed (check `gateway.auth.token` in config)
5. Run `openclaw cron list` to verify job is loaded
6. Optionally `openclaw cron run daily-self-audit --force` to test

## Additional Issue Found

The `npx openclaw` CLI has deprecated dependencies with known vulnerabilities:
- tar@6.2.1 — "widely publicized security vulnerabilities"  
- glob@10.5.0 — "widely publicized security vulnerabilities"

Carlos flagged this as a security concern. Consider:
- Opening a GitHub issue on openclaw/openclaw about vulnerable transitive deps
- Checking if newer versions fix this

## Context

Carlos wants Eddie to proactively audit for:
1. Security threats (OpenClaw is a potential target)
2. Routine maintenance (file cleanup, memory consolidation)

Job delivers results to Carlos's Telegram at 3am ET daily.

---

## Feature Request: Give Eddie Cron Autonomy

**Issue:** Eddie cannot create cron jobs directly. He's sandboxed and can't write to host files.

**Current state:**
- Eddie's available tools: read, write, edit, exec, process, sessions_*, session_status, image
- All file operations are sandboxed to `~/.openclaw/workspace/`
- Cron jobs live at `~/.openclaw/cron/jobs.json` on the host (outside sandbox)
- No cron.* tools exposed to agents

**What works (manual):**
- Direct editing of `~/.openclaw/cron/jobs.json` via nano/vim
- Restart `openclaw.service` to reload
- No CLI needed (CLI has auth issues and vulnerable deps anyway)

**Options to give Eddie autonomy:**

**Option A (Preferred):** Expose Gateway cron APIs as agent tools
- `cron.add`, `cron.list`, `cron.remove`, `cron.run`
- Gateway already has these APIs internally
- Just need to wire them up as callable tools

**Option B:** Mount cron jobs file into sandbox
- Give Eddie write access to `~/.openclaw/cron/jobs.json`
- Eddie edits directly, Gateway picks up changes

**Option C:** Create a helper script on host
- Script at `/home/openclaw/homebrew/cron_manager.py`
- Eddie calls via exec, script modifies jobs.json
- Requires elevated exec permissions

**Why it matters:**
- Self-management is a core Eddie capability
- Cron jobs are essential for proactive behavior (audits, briefings, reminders)
- Current setup requires Carlos to SSH into droplet for every new cron job
- This creates unnecessary friction

**Priority:** P1 — Blocks Eddie's autonomy for scheduled tasks
