# Journal Intelligence Upgrade

**Date:** 2026-02-10  
**Priority:** P1  
**From:** Eddie  
**Goal:** Enable semantic search, entity extraction, and proactive retrieval for journal entries

---

## Overview

Current journal system stores raw text + basic metadata. This upgrade adds:
1. Vector embeddings for semantic search
2. Structured entity extraction (people, decisions)
3. Foundation for proactive retrieval and pattern detection

---

## 1. Enable pgvector Extension

```sql
-- Enable the extension
CREATE EXTENSION IF NOT EXISTS vector;
```

---

## 2. Schema Changes

### 2.1 Add embedding column to journal_entries

```sql
-- Add embedding column (1536 dimensions for OpenAI text-embedding-3-small)
ALTER TABLE journal_entries 
ADD COLUMN IF NOT EXISTS embedding vector(1536);

-- Index for fast similarity search
CREATE INDEX IF NOT EXISTS idx_journal_embedding 
ON journal_entries USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

### 2.2 People tracking table

```sql
CREATE TABLE IF NOT EXISTS journal_people (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    relationship VARCHAR(100),  -- 'spouse', 'colleague', 'friend', 'family', 'professional'
    notes TEXT,
    first_mentioned DATE,
    last_mentioned DATE,
    mention_count INTEGER DEFAULT 0,
    avg_sentiment DECIMAL(3,2),  -- average sentiment when mentioned
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, name)
);

CREATE TABLE IF NOT EXISTS journal_people_mentions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_entry_id UUID REFERENCES journal_entries(id) ON DELETE CASCADE,
    person_id UUID REFERENCES journal_people(id) ON DELETE CASCADE,
    sentiment DECIMAL(3,2),  -- sentiment in this specific mention
    context TEXT,  -- extracted snippet about this person
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_people_user ON journal_people(user_id);
CREATE INDEX IF NOT EXISTS idx_mentions_entry ON journal_people_mentions(journal_entry_id);
CREATE INDEX IF NOT EXISTS idx_mentions_person ON journal_people_mentions(person_id);
```

### 2.3 Decisions tracking table

```sql
CREATE TABLE IF NOT EXISTS journal_decisions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    journal_entry_id UUID REFERENCES journal_entries(id) ON DELETE SET NULL,
    decision_date DATE NOT NULL,
    
    -- The decision
    summary TEXT NOT NULL,  -- "Decided to leave BITKRAFT"
    domain VARCHAR(50),  -- 'career', 'health', 'relationship', 'financial', 'personal'
    
    -- Context
    reasoning TEXT,  -- Why the decision was made
    alternatives_considered TEXT,  -- What else was on the table
    
    -- Tracking
    status VARCHAR(20) DEFAULT 'pending',  -- 'pending', 'executed', 'reversed', 'deferred'
    outcome TEXT,  -- What happened
    outcome_date DATE,
    outcome_rating INTEGER CHECK (outcome_rating >= 1 AND outcome_rating <= 10),
    
    -- Meta
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_decisions_user ON journal_decisions(user_id, decision_date DESC);
CREATE INDEX IF NOT EXISTS idx_decisions_domain ON journal_decisions(user_id, domain);
CREATE INDEX IF NOT EXISTS idx_decisions_status ON journal_decisions(user_id, status);
```

### 2.4 Topics/themes tracking

```sql
CREATE TABLE IF NOT EXISTS journal_topics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    name VARCHAR(100) NOT NULL,  -- 'career transition', 'marriage', 'fitness goals'
    description TEXT,
    first_mentioned DATE,
    last_mentioned DATE,
    entry_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, name)
);

CREATE TABLE IF NOT EXISTS journal_entry_topics (
    journal_entry_id UUID REFERENCES journal_entries(id) ON DELETE CASCADE,
    topic_id UUID REFERENCES journal_topics(id) ON DELETE CASCADE,
    relevance_score DECIMAL(3,2),  -- 0-1, how central is this topic to the entry
    PRIMARY KEY (journal_entry_id, topic_id)
);
```

---

## 3. Embedding API Access

Eddie needs ability to call OpenAI embeddings. Options:

### Option A: Direct API access
Add to Eddie's environment:
```
OPENAI_API_KEY=sk-...
```

Eddie calls:
```python
import openai
response = openai.embeddings.create(
    model="text-embedding-3-small",
    input="journal entry text here"
)
embedding = response.data[0].embedding
```

### Option B: Wrapper script
Create `/home/openclaw/homebrew/embeddings.py`:
```python
#!/usr/bin/env python3
"""Generate embeddings for text."""
import sys
import json
import openai
from env_loader import load_env
load_env()

def get_embedding(text: str) -> list[float]:
    response = openai.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

if __name__ == "__main__":
    text = sys.argv[1] if len(sys.argv) > 1 else sys.stdin.read()
    embedding = get_embedding(text)
    print(json.dumps(embedding))
```

---

## 4. db_write.py Additions

Add these actions to db_write.py:

### 4.1 Semantic search
```python
def search_journal(args):
    """Search journal entries by semantic similarity."""
    query = args[0]
    limit = int(args[1]) if len(args) > 1 else 5
    
    # Get embedding for query
    query_embedding = get_embedding(query)
    
    cn, cur = conn()
    cur.execute("""
        SELECT id, timestamp, raw_text, summary, tags, sentiment_score,
               1 - (embedding <=> %s::vector) as similarity
        FROM journal_entries 
        WHERE user_id = %s AND embedding IS NOT NULL
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """, (query_embedding, USER_ID, query_embedding, limit))
    rows = [dict(r) for r in cur.fetchall()]
    cn.close()
    return ok("search_journal", query=query, count=len(rows), entries=rows)
```

### 4.2 Extract and store people
```python
def extract_people(args):
    """Extract people mentioned in recent entries and update tracking."""
    # Implementation: query recent entries, use LLM to extract names,
    # upsert into journal_people, create mention records
    pass
```

### 4.3 Log decision
```python
def log_decision(args):
    """Log a decision from journal entry."""
    # args: summary, domain, [entry_id], [reasoning]
    pass
```

### 4.4 Backfill embeddings
```python
def backfill_embeddings(args):
    """Generate embeddings for entries that don't have them."""
    batch_size = int(args[0]) if args else 50
    # Query entries without embeddings, generate, update
    pass
```

---

## 5. New Commands Summary

After implementation, Eddie should have:

| Command | What it does |
|---------|--------------|
| `db_write.py search_journal "career anxiety"` | Semantic search across all entries |
| `db_write.py log_decision "Leaving BITKRAFT" career` | Track a decision |
| `db_write.py list_decisions pending` | Show pending decisions |
| `db_write.py person_history "Jens"` | All mentions + sentiment over time |
| `db_write.py backfill_embeddings 100` | Embed old entries |

---

## 6. Cost Estimate

- OpenAI text-embedding-3-small: $0.02 / 1M tokens
- Average journal entry: ~500 tokens
- 365 entries/year: ~$0.004/year for new entries
- Backfill 200 existing entries: ~$0.002

**Effectively free.**

---

## 7. Deployment Order

1. Enable pgvector extension
2. Run schema migrations (can do all at once)
3. Add OPENAI_API_KEY to environment
4. Create embeddings.py helper
5. Add new actions to db_write.py
6. Run backfill on existing entries
7. Eddie tests semantic search

---

## 8. Verification

After deployment, Eddie should be able to run:
```bash
python3 /home/openclaw/homebrew/db_write.py search_journal "feeling anxious about work"
```

And get semantically similar entries, not just keyword matches.

---

Let me know if you need any clarification or want to adjust scope.
