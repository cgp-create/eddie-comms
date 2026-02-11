# P0: Audio Transcription Regression Fix

**Priority:** 0.5 (Critical - regression blocking voice notes)  
**Status:** Blocking  
**From:** Eddie  
**Date:** 2026-02-09

---

## Problem

Voice notes from Carlos are NOT being transcribed. Eddie receives raw OGG files instead of text.

**⚠️ THIS IS A REGRESSION** - Audio transcription worked before. Something broke.

---

## Evidence

- Carlos sent voice message: `file_26---c6687ea0-0d36-4bd0-9be1-c7aabf68c4c5.ogg` (79KB)
- Eddie received binary audio data instead of transcript
- Eddie tested Gemini fallback via agent-level `image` tool - returns "Unsupported media type: audio"
- No agent-level tools available for audio transcription
- Carlos confirms this worked in previous sessions

---

## Expected Behavior

OpenClaw should transcribe voice notes via `tools.media.audio` (Deepgram/Whisper) **before messages reach the agent**.

---

## Fix Needed

### 1. Find and Fix What Broke

- Check gateway config for `tools.media.audio` configuration
- Verify Deepgram API key or Whisper installation
- Test transcription with waiting voice note (79KB OGG file at path above)
- Confirm transcripts appear in prompt instead of raw audio

### 2. Add Router-Level Fallback (Carlos's Request)

**Architecture:** Gateway/router should try primary transcription first, auto-fallback to Gemini 2.0 Flash if primary fails.

**Why Gemini:** Supports audio natively, Carlos already has API key configured.

**Benefit:** Resilient transcription - if one method fails, fallback kicks in automatically.

**Location:** This should happen at gateway/router level BEFORE message reaches agent.

**Implementation concept:**
```
Inbound voice note → 
  Try: Deepgram/Whisper → 
    Success? → Send transcript to agent
    Fail? → Try: Gemini 2.0 Flash audio → 
      Success? → Send transcript to agent
      Fail? → Send error message to agent
```

**Carlos's setup:** Already has Gemini key. Just needs router-level fallback logic.

---

## Test File

**Path:** `file_26---c6687ea0-0d36-4bd0-9be1-c7aabf68c4c5.ogg` (79KB)

After fix, Carlos can resend this or send a new voice note to verify transcription works.

---

## Documentation Reference

- `/opt/openclaw/docs/providers/deepgram.md`
- Check gateway logs for transcription errors

---

## Success Criteria

✅ Carlos sends voice note  
✅ Eddie receives text transcript in prompt  
✅ If primary transcription fails, Gemini fallback activates automatically  
✅ No manual intervention required for future voice notes
