# Multi-User Data Separation

**Priority:** P1 (prerequisite for Melissa onboarding)
**Date:** 2026-02-11
**From:** Eddie

## Context

Melissa will start using Eddie for her own journal and fitness tracking. Carlos requires strict data separation — his journal/fitness data must never leak to her session, and vice versa.

## Current State (Unknown)

I cannot verify from sandbox whether personal tables have `user_id` columns. Need Claude to check:

```sql
SELECT column_name 
FROM information_schema.columns 
WHERE table_name IN ('journal_entries', 'workouts', 'body_measurements', 'fitness_profiles')
ORDER BY table_name, ordinal_position;
```

## Requirements

**1. Schema Changes (if needed)**

Every personal data table needs:
```sql
ALTER TABLE journal_entries ADD COLUMN user_id UUID NOT NULL DEFAULT 'carlos-uuid';
ALTER TABLE workouts ADD COLUMN user_id UUID NOT NULL DEFAULT 'carlos-uuid';
ALTER TABLE body_measurements ADD COLUMN user_id UUID NOT NULL DEFAULT 'carlos-uuid';
-- etc for any other personal tables
```

**2. Users Table (if doesn't exist)**

```sql
CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    telegram_id BIGINT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    email TEXT,
    timezone TEXT DEFAULT 'America/New_York',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Seed Carlos
INSERT INTO users (telegram_id, name, email) 
VALUES (1832767482, 'Carlos', 'cacoguedes@gmail.com');
```

**3. Query Filtering**

All db_write.py commands that touch personal data must:
- Accept `user_id` parameter (or derive from session context)
- Filter queries: `WHERE user_id = %s`
- Never return cross-user data

Commands to audit:
- `recent_journal`
- `recent_workouts`
- `search_journal`
- Any fitness/body measurement queries

**4. Session-to-User Mapping**

OpenClaw sessions have `telegram_id` in delivery context. The flow:
1. Eddie receives message with sender's Telegram ID
2. Look up `user_id` from `users` table by `telegram_id`
3. Pass `user_id` to all DB queries

Question for Claude: How does Eddie currently know who's messaging? Is `telegram_id` available in session context that I can use?

**5. Migration Plan**

1. Add `users` table
2. Insert Carlos as first user
3. Add `user_id` column to personal tables (default to Carlos's UUID)
4. Update all existing rows to Carlos's user_id
5. Add NOT NULL constraint
6. Update db_write.py to require user_id on personal queries
7. Test with Carlos's data
8. Add Melissa to users table when ready

## Shared vs Personal Data

**Personal (requires user_id filtering):**
- journal_entries
- workouts
- body_measurements
- fitness_profiles

**Shared (accessible to both):**
- restaurants
- contacts

**Personal (Carlos-only, but not journal/fitness):**
- positions / position_research (portfolio data)
- decisions (investment decision log)
- briefings (portfolio briefings)

## Open Questions

1. Is `telegram_id` available to Eddie at runtime? How do I access it?
2. Do we need separate fitness_profiles per user, or is that overkill?

## Acceptance Criteria

- [ ] Carlos's journal queries return only Carlos's entries
- [ ] Melissa's journal queries return only Melissa's entries
- [ ] No way to accidentally query cross-user data
- [ ] Existing Carlos data preserved with his user_id
- [ ] Eddie can determine current user from session context

---

**Status:** Spec complete, needs Claude implementation
