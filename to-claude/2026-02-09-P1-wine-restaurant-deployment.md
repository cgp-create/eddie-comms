# Wine & Restaurant Database Deployment

**Date:** 2026-02-09 22:00 UTC  
**Priority:** P1  
**Status:** Ready for deployment  
**Requester:** Carlos  
**Prepared by:** Eddie

---

## Summary

Eddie has created a complete SQL schema for wine tracking (champagne, red, white) and restaurant tracking. All files are ready for deployment. Carlos approved the design. Need Claude to deploy schema and data to production database.

---

## What's Been Created

### Files in `/home/openclaw/.openclaw/workspace/migrations/`:

1. **`003_wine_restaurant_schema.sql`** (6.7KB)
   - Creates 5 new tables
   - wine_bottles, wine_preferences, wine_occasions, restaurants, restaurant_visits
   - Includes all indexes

2. **`003_champagne_data.sql`** (9.4KB)
   - Migrates 19 champagne bottles from markdown to SQL
   - 1 preference profile
   - 7 occasion mappings

3. **`migrate_champagne_data.py`** (14KB)
   - Python script that generated the data SQL
   - Can be reused for future migrations
   - Includes parser for markdown format

### Documentation:

4. **`wine-restaurant-schema-design.md`** (10.9KB)
   - Complete design rationale
   - Example queries
   - Benefits analysis

---

## Deployment Steps

### Step 1: Deploy Schema

**Run on production database:**

```bash
psql -h <DB_HOST> -U <DB_USER> -d <DB_NAME> \
  < /home/openclaw/.openclaw/workspace/migrations/003_wine_restaurant_schema.sql
```

**Or via Python:**

```bash
python3 << 'EOF'
import sys
sys.path.insert(0, '/home/openclaw/homebrew')
from eddie_config import DB_CONFIG
import psycopg2

with open('/home/openclaw/.openclaw/workspace/migrations/003_wine_restaurant_schema.sql') as f:
    schema_sql = f.read()

conn = psycopg2.connect(**DB_CONFIG)
cur = conn.cursor()
cur.execute(schema_sql)
conn.commit()
conn.close()
print("✅ Schema deployed")
EOF
```

**Verification:**

```bash
python3 -c "
import sys
sys.path.insert(0, '/home/openclaw/homebrew')
from eddie_config import DB_CONFIG
import psycopg2

conn = psycopg2.connect(**DB_CONFIG)
cur = conn.cursor()
cur.execute(\"SELECT tablename FROM pg_tables WHERE schemaname='public' AND (tablename LIKE 'wine_%' OR tablename LIKE 'restaurant%') ORDER BY tablename\")
tables = [row[0] for row in cur.fetchall()]
conn.close()

expected = ['restaurants', 'restaurant_visits', 'wine_bottles', 'wine_occasions', 'wine_preferences']
print('Tables created:')
for t in tables:
    print(f'  ✅ {t}')

if set(tables) == set(expected):
    print('\n✅ All 5 tables created successfully')
else:
    missing = set(expected) - set(tables)
    print(f'\n❌ Missing tables: {missing}')
"
```

---

### Step 2: Load Champagne Data

**Run on production database:**

```bash
psql -h <DB_HOST> -U <DB_USER> -d <DB_NAME> \
  < /home/openclaw/.openclaw/workspace/migrations/003_champagne_data.sql
```

**Or via Python:**

```bash
python3 << 'EOF'
import sys
sys.path.insert(0, '/home/openclaw/homebrew')
from eddie_config import DB_CONFIG
import psycopg2

with open('/home/openclaw/.openclaw/workspace/migrations/003_champagne_data.sql') as f:
    data_sql = f.read()

conn = psycopg2.connect(**DB_CONFIG)
cur = conn.cursor()
cur.execute(data_sql)
conn.commit()
conn.close()
print("✅ Champagne data migrated")
EOF
```

**Verification:**

```bash
python3 -c "
import sys
sys.path.insert(0, '/home/openclaw/homebrew')
from eddie_config import DB_CONFIG
import psycopg2

conn = psycopg2.connect(**DB_CONFIG)
cur = conn.cursor()

# Count bottles
cur.execute('SELECT count(*) FROM wine_bottles')
bottles = cur.fetchone()[0]

# Count preferences
cur.execute('SELECT count(*) FROM wine_preferences')
prefs = cur.fetchone()[0]

# Count occasions
cur.execute('SELECT count(*) FROM wine_occasions')
occs = cur.fetchone()[0]

conn.close()

print(f'✅ wine_bottles: {bottles} (expected 19)')
print(f'✅ wine_preferences: {prefs} (expected 1)')
print(f'✅ wine_occasions: {occs} (expected 7)')

if bottles == 19 and prefs == 1 and occs == 7:
    print('\n✅ Data migration successful')
else:
    print('\n⚠️  Counts don''t match expected values')
"
```

---

### Step 3: Test Queries

**Run sample queries to verify data integrity:**

```bash
python3 << 'EOF'
import sys
sys.path.insert(0, '/home/openclaw/homebrew')
from eddie_config import DB_CONFIG
import psycopg2

conn = psycopg2.connect(**DB_CONFIG)
cur = conn.cursor()

print("Sample Queries:")
print("-" * 80)

# Elite tier bottles
print("\n1. Elite Tier Champagnes:")
cur.execute("SELECT producer, name, rating FROM wine_bottles WHERE tier='elite' AND status='tried' ORDER BY rating DESC")
for row in cur.fetchall():
    print(f"   {row[0]} - {row[1]} (Rating: {row[2]})")

# Daily drivers
print("\n2. Daily Drivers:")
cur.execute("SELECT producer, name, price_usd FROM wine_bottles WHERE tier='daily_driver' ORDER BY price_usd")
for row in cur.fetchall():
    print(f"   {row[0]} - {row[1]} (${row[2]})")

# Bucket list
print("\n3. Bucket List:")
cur.execute("SELECT producer, name FROM wine_bottles WHERE status='bucket_list'")
for row in cur.fetchall():
    print(f"   {row[0]} - {row[1]}")

# Occasion mapping
print("\n4. Price Tiers by Occasion:")
cur.execute("SELECT occasion_type, price_min, price_max FROM wine_occasions ORDER BY price_min")
for row in cur.fetchall():
    print(f"   {row[0]}: ${row[1]}-${row[2]}")

conn.close()
print("\n✅ All queries executed successfully")
EOF
```

---

## What This Enables

### Wine Tracking

Carlos can now:
- Query champagnes by price range, rating, tier
- Track what he's tried vs bucket list
- Get recommendations by occasion
- Record tasting notes
- Extend to red wine, white wine when ready

### Example Queries Now Possible:

```sql
-- Show bottles under $150 sorted by rating
SELECT producer, name, price_usd, rating
FROM wine_bottles
WHERE price_usd <= 150 AND status='tried'
ORDER BY rating DESC;

-- What should I buy for anniversary?
SELECT producer, name, price_usd
FROM wine_bottles
WHERE price_usd BETWEEN 250 AND 350
  AND rating >= 8
ORDER BY rating DESC;

-- All rosé champagnes
SELECT producer, name, rating, price_usd
FROM wine_bottles
WHERE sub_category='rosé'
ORDER BY rating DESC;
```

---

## Future Extensions

### Restaurants (Schema Ready, No Data Yet)

Once restaurants are tracked:
- Log visits with dishes, ratings
- Track "want to try" vs "been"
- Get recommendations by cuisine/occasion
- Remember what was ordered last time

### CLI Tools (Eddie Will Create Next)

```bash
python3 eddie_wine.py list --max-price 150
python3 eddie_wine.py recommend --occasion anniversary
python3 eddie_wine.py bucket_list
python3 eddie_restaurants.py recommend --cuisine Italian
```

---

## Risk Assessment

**Low Risk:**

1. ✅ New tables (not modifying existing schema)
2. ✅ No data loss (markdown still exists as backup)
3. ✅ Reversible (can drop tables if needed)
4. ✅ Tested SQL syntax
5. ✅ Carlos approved design

**Rollback Plan:**

If needed:
```sql
DROP TABLE IF EXISTS restaurant_visits CASCADE;
DROP TABLE IF EXISTS restaurants CASCADE;
DROP TABLE IF EXISTS wine_occasions CASCADE;
DROP TABLE IF EXISTS wine_preferences CASCADE;
DROP TABLE IF EXISTS wine_bottles CASCADE;
```

---

## Next Steps After Deployment

1. **Carlos verifies** - spot check data matches markdown
2. **Eddie creates CLI tools** - wine querying/adding commands
3. **Carlos starts using** - add new bottles, query existing
4. **Extend to restaurants** - when Carlos wants to track
5. **Add red/white wine** - same schema, just category='red'

---

## Questions for Claude

None - deployment is straightforward. Just run the two SQL files and verify counts.

---

## Timeline

**Total deployment time:** ~5 minutes
- Schema creation: 1 min
- Data loading: 1 min
- Verification: 3 min

**Eddie's next work:** CLI tools (1-2 hours after deployment)

---

## Files Summary

```
/home/openclaw/.openclaw/workspace/migrations/
├── 003_wine_restaurant_schema.sql  (6.7KB) - CREATE TABLES + INDEXES
├── 003_champagne_data.sql          (9.4KB) - INSERT DATA
└── migrate_champagne_data.py       (14KB)  - Migration script (reusable)

/home/openclaw/.openclaw/workspace/
└── wine-restaurant-schema-design.md (10.9KB) - Design doc
```

---

**Status:** ✅ Ready for deployment  
**Blocker:** None - just needs SQL execution  
**Risk:** Low  
**Estimated time:** 5 minutes

**Recommendation:** Deploy when convenient. No urgency, but unlocks wine tracking functionality Carlos approved.
