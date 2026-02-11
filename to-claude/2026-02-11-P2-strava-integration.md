# Strava Integration Research

**Priority:** P2
**Date:** 2026-02-11
**From:** Eddie

## Context

Carlos wants Strava integration for workout tracking (simpler than Garmin). This replaces the earlier Garmin research.

## Strava API Overview

**Base URL:** `https://www.strava.com/api/v3`
**Auth:** OAuth 2.0 (requires user authorization)
**Rate Limits:** 100 requests/15 min, 1000 requests/day (free tier)

## Key Endpoints

**Activities:**
- `GET /athlete/activities` — List athlete activities (paginated)
- `GET /activities/{id}` — Get activity details
- `POST /activities` — Create manual activity

**Athlete:**
- `GET /athlete` — Get authenticated athlete profile
- `GET /athlete/stats` — Get athlete stats (totals, recent)

## OAuth Flow

1. Register app at https://www.strava.com/settings/api
2. Get `client_id` and `client_secret`
3. Redirect user to authorize: `https://www.strava.com/oauth/authorize?client_id=XXX&redirect_uri=YYY&response_type=code&scope=activity:read_all`
4. Exchange code for access token + refresh token
5. Store refresh token, use to get new access tokens

**Scopes needed:**
- `activity:read_all` — Read all activities (including private)
- `profile:read_all` — Read profile info

## Data Available Per Activity

```json
{
  "id": 12345,
  "name": "Morning Ride",
  "type": "Ride",
  "sport_type": "Ride",
  "start_date": "2026-02-11T07:00:00Z",
  "elapsed_time": 3600,
  "moving_time": 3400,
  "distance": 25000,
  "average_speed": 7.35,
  "max_speed": 12.5,
  "average_heartrate": 145,
  "max_heartrate": 175,
  "calories": 650,
  "suffer_score": 78
}
```

## Implementation Plan

**Phase 1: OAuth Setup**
- Create Strava app, get credentials
- Build OAuth flow (similar to Google Calendar)
- Store tokens in `.env` or secure storage

**Phase 2: Sync Script**
- `strava_sync.py` — Fetch recent activities
- Map Strava activity types to Eddie's workout categories
- Store in `workouts` table

**Phase 3: Auto-Sync (Optional)**
- Webhook subscription for real-time activity updates
- Or: cron job to poll every few hours

## Comparison to Garmin

| Feature | Strava | Garmin |
|---------|--------|--------|
| OAuth | Standard | Complex |
| Rate limits | Generous | Tight |
| Data richness | Good | Better (HRV, sleep) |
| Setup complexity | Low | High |
| Carlos preference | ✅ | ❌ |

## What Claude Needs to Build

1. **strava_oauth.py** — Token management (similar to gcal.py pattern)
2. **strava_sync.py** — Activity fetch + database storage
3. **Cron job** — Daily sync of new activities
4. **DB migration** — Ensure `workouts` table can store Strava activity IDs

## Open Questions for Carlos

- Which Strava account to connect? (Assuming personal)
- Sync frequency preference? (Daily should be fine)
- Want heart rate / suffer score data captured?

---

**Status:** Research complete, ready for implementation when prioritized
