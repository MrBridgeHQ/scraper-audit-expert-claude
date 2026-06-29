# Quality and Consistency Tests — Concrete Queries

This file gives the **concrete tests** for Dimensions 2 (Quality) and 3 (Consistency) of `audit-framework.md`. Each test:
- Has a **runnable query** (jq, SQL, bash)
- Has a **pass/fail criterion**
- Has a **finding template** if it fails

Use these during Phase 2 of the audit. Branches on runtime (Apify dataset vs. Postgres) where needed.

---

## Quality tests

### Q1 — Per-field null rate

**Question:** What % of records are missing each field?

**Apify (jq on dataset items):**

```bash
apify dataset get <datasetId> --limit=10000 --json > /tmp/items.json

# Per-field null/empty rate
jq '[.[]] | length as $total | [(.[0] | keys[]) as $k | {field: $k, pct_null: (100.0 * ([.[] | select(.[$k] == null or .[$k] == "" or .[$k] == [])] | length) / $total)}]' /tmp/items.json
```

**Standalone (Postgres):**

```sql
SELECT
  100.0 * COUNT(*) FILTER (WHERE name IS NULL OR name = '') / NULLIF(COUNT(*), 0) AS pct_null_name,
  100.0 * COUNT(*) FILTER (WHERE price IS NULL) / NULLIF(COUNT(*), 0) AS pct_null_price,
  100.0 * COUNT(*) FILTER (WHERE currency IS NULL OR currency = '') / NULLIF(COUNT(*), 0) AS pct_null_currency,
  100.0 * COUNT(*) FILTER (WHERE availability IS NULL) / NULLIF(COUNT(*), 0) AS pct_null_availability
FROM scraped_records
WHERE created_at > NOW() - INTERVAL '7 days';
```

**Group by source/locale for hidden localization bugs:**

```sql
SELECT
  country_code,
  100.0 * COUNT(*) FILTER (WHERE price IS NULL) / NULLIF(COUNT(*), 0) AS pct_null_price
FROM scraped_records
WHERE created_at > NOW() - INTERVAL '7 days'
GROUP BY country_code
ORDER BY pct_null_price DESC;
```

**Pass criterion:** for fields declared non-nullable, pct_null ≤ 1%. For nullable fields, pct_null < documented baseline.

**Finding template (Q1 fail):**
```
[SEV-N][S][quality] `<field>` null rate is X% (baseline expected <Y%)
  Evidence: <SQL/jq query output>
  Group breakdown: <if localized>: pct_null on locale `<L>` is Z%
  Impact: <% of frontend renders affected>
  Fix: cross-ref your selector-strategy doctrine § "Layered fallbacks"
```

### Q2 — Schema conformance

**Question:** Do all records validate against the declared schema?

**Apify:**

```bash
# Assuming schema is in .actor/dataset_schema.json
jq -c '.[]' /tmp/items.json | while read item; do
  echo "$item" | jsonschema -i /dev/stdin .actor/dataset_schema.json 2>&1
done | grep -v "valid" | wc -l   # invalid count
```

**Standalone (TypeScript with Zod):**

```ts
// Add to backend admin endpoint
import { ProductSchema } from '@shared/schemas';
import { db } from '../db';

const recent = await db.query('SELECT * FROM scraped_records WHERE created_at > NOW() - INTERVAL \'7 days\' LIMIT 10000');
const invalid = recent.rows.filter((r) => !ProductSchema.safeParse(r).success);
console.log(`${invalid.length} of ${recent.rows.length} records fail schema validation`);
console.log(invalid.slice(0, 5).map((r) => ProductSchema.safeParse(r).error.errors));
```

**Pass criterion:** ≥99% records validate.

**Finding template (Q2 fail):**
```
[SEV-4][S][quality] X% of records fail schema validation
  Top violation patterns:
    - `price` is string where number expected (N records)
    - `currency` enum violation: "$" instead of "USD" (M records)
    - `imageUrl` is empty string where url expected (K records)
  Fix: cross-ref a Crawlee-template reference § "Defensive extraction"
```

### Q3 — Accuracy sample (manual, but structured)

**Question:** Does the extracted value match the live source?

**Procedure:**
1. Sample 100 random records: `SELECT * FROM scraped_records TABLESAMPLE BERNOULLI(1) LIMIT 100;` or `jq '.[] | select(. as $r | (1000 * (now | sin | abs) | floor) % 100 == 0)' /tmp/items.json | head -100`
2. For each, open the source URL and verify each field
3. Tag each field per record as: `OK`, `MISSING` (null when source had value), `WRONG` (value ≠ source), `STALE` (source changed since scrape), `FORMAT` (right value, wrong type)
4. Tally per-field error rate

**Spreadsheet template:**

| record_id | url | name | price | currency | availability | image_url | rating |
|---|---|---|---|---|---|---|---|
| 001 | ... | OK | OK | WRONG (USD vs source GBP) | OK | OK | MISSING |
| 002 | ... | OK | FORMAT (string vs number) | OK | OK | STALE | OK |
| ... | | | | | | | |

**Pass criterion:** per-field error rate < 5% (excluding STALE which is acceptable noise).

**Finding template (Q3 fail):**
```
[SEV-4][S][quality] `<field>` accuracy: X% wrong / Y% missing on N=100 sample
  Top error pattern: <e.g. "GBP listings extracted as USD because currency symbol parsing fails">
  Sample bad records: <list 3 ids + their wrong values>
  Fix: cross-ref your selector-strategy doctrine § "Normalization"
```

### Q4 — Normalization audit

**Question:** Are fields normalized to a canonical form?

**Currency uniformity:**

```sql
-- Should be ISO 4217 codes only. Any other values = parsing leak.
SELECT currency, COUNT(*) FROM scraped_records GROUP BY currency ORDER BY 2 DESC LIMIT 20;
-- Expected: USD, EUR, GBP, etc.
-- Bad: $, £, €, "USD$", null, ""
```

**Price as number not string:**

```sql
SELECT 100.0 * COUNT(*) FILTER (WHERE pg_typeof(price) = 'text') / COUNT(*) AS pct_string_price FROM scraped_records;
-- Expected 0%
```

**Date format consistency:**

```sql
-- Looking for ISO 8601 strings vs. locale strings vs. timestamps
SELECT scraped_at, pg_typeof(scraped_at) FROM scraped_records LIMIT 10;
-- All should be timestamptz, not text
```

**URL absoluteness:**

```sql
SELECT COUNT(*) FILTER (WHERE image_url LIKE '/%' OR image_url LIKE '//%') AS relative_urls FROM scraped_records;
-- Expected 0 — all URLs should be absolute (https://...)
```

**String trimming:**

```sql
SELECT COUNT(*) FILTER (WHERE name <> TRIM(name)) AS untrimmed_names FROM scraped_records;
-- Expected 0
```

**HTML entities in strings:**

```sql
SELECT COUNT(*) FILTER (WHERE name LIKE '%&amp;%' OR name LIKE '%&quot;%' OR name LIKE '%&#%') AS entity_polluted FROM scraped_records;
-- Expected 0
```

**Pass criterion:** all normalization checks pass at >99%.

**Finding template (Q4 fail):**
```
[SEV-3][S][quality] Normalization issue: <type>
  Evidence: <query output>
  Impact: <downstream effect — search broken? sorting wrong? frontend rendering bad?>
  Fix: add normalization step in extractor; cross-ref a Crawlee-template reference § `src/extractors/productCard.ts`
```

### Q5 — PII / safety leak check

**Question:** Are any records leaking auth tokens, session IDs, or user PII?

```sql
-- Look for token patterns in textual fields
SELECT id, name FROM scraped_records
WHERE name ~* '(bearer|token|session|jwt|api[_-]?key|password)' OR
      name ~* '[a-zA-Z0-9]{40,}'  -- long opaque strings
LIMIT 20;

-- Check for email patterns (PII concern if not expected)
SELECT id, description FROM scraped_records
WHERE description ~* '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'
LIMIT 20;
```

**Pass criterion:** 0 matches.

**Finding template (Q5 fail):** 🔴 SEV-5 always. Stop the scraper, redact, then deploy fix.

### Q6 — Junk record detection

**Question:** Are error pages / captchas / empty objects being persisted as valid records?

```sql
-- Empty objects (all fields null)
SELECT id FROM scraped_records
WHERE name IS NULL AND price IS NULL AND url IS NULL
LIMIT 20;

-- Captcha indicators
SELECT id, name FROM scraped_records
WHERE name ILIKE '%captcha%' OR name ILIKE '%cloudflare%' OR name ILIKE '%checking your browser%'
LIMIT 20;

-- Generic "error page" titles
SELECT id, name FROM scraped_records
WHERE name IN ('Page Not Found', '404', 'Error', '500 Internal Server Error', 'Access Denied')
LIMIT 20;
```

**Pass criterion:** 0 matches.

**Finding template (Q6 fail):** 🔴 SEV-4. Add a `hasUsefulData()` style guard in the parser before persisting; cross-ref your error-handling doctrine § "EXTRACTION_FAILED".

---

## Consistency tests

### C1 — Cross-run stability

**Question:** Same URL scraped twice = same data?

**Procedure:**
1. Pick 20 stable URLs (products unlikely to change in 24h — popular catalog items, established merchants)
2. Scrape them today; persist with `audit_run = 1`
3. Scrape them again in 24h; persist with `audit_run = 2`
4. Diff per field

**Query:**

```sql
WITH paired AS (
  SELECT url,
    MAX(name) FILTER (WHERE audit_run = 1) AS name_t1,
    MAX(name) FILTER (WHERE audit_run = 2) AS name_t2,
    MAX(price) FILTER (WHERE audit_run = 1) AS price_t1,
    MAX(price) FILTER (WHERE audit_run = 2) AS price_t2
  FROM scraped_records WHERE audit_run IN (1, 2)
  GROUP BY url
)
SELECT
  COUNT(*) FILTER (WHERE name_t1 <> name_t2) AS name_changed,
  COUNT(*) FILTER (WHERE price_t1 <> price_t2) AS price_changed,
  COUNT(*) AS total
FROM paired;
```

**Pass criterion:**
- `name` should change 0% (product names don't change in 24h)
- `price` may legitimately change for live-market sources (volatile, frequently-repriced catalogs); should be stable for static catalog sources

**Finding template (C1 fail):**
```
[SEV-4][S][consistency] `<field>` changes between runs at X% rate (expected 0%)
  Sample diffs: <3 examples — same URL, different value>
  Likely cause: <selector returns one of multiple matching elements, ordering nondeterministic>
  Fix: cross-ref your selector-strategy doctrine § "Layered fallbacks"
        — use `.first()`, anchor on stable element, normalize before persistence
```

### C2 — Dedup correctness

**Question:** Same record persisted multiple times?

```sql
-- For natural-key dedup
SELECT name, variant, brand, COUNT(*) AS dup_count
FROM scraped_records
GROUP BY name, variant, brand
HAVING COUNT(*) > 1
ORDER BY dup_count DESC
LIMIT 50;

-- For URL-based dedup
SELECT url, COUNT(*) AS dup_count
FROM scraped_records
GROUP BY url
HAVING COUNT(*) > 1
ORDER BY dup_count DESC
LIMIT 50;
```

**Pass criterion:** ≤1% duplicate rate. Some duplicates are legitimate (re-scrapes for freshness); rate-spike indicates dedup logic bug.

**Finding template (C2 fail):**
```
[SEV-3][M][consistency] X% duplicate rate on natural key (name, variant, brand)
  Top duplicates: <list 3 examples with counts>
  Likely cause: <e.g. case-insensitive match missing, whitespace not normalized before insert>
  Fix: add unique constraint with normalized columns; cross-ref a Crawlee-template reference
```

### C3 — Schema drift over time

**Question:** Has a field's null rate / type / value distribution shifted?

```sql
-- Null rate trend by week
SELECT
  DATE_TRUNC('week', created_at) AS week,
  100.0 * COUNT(*) FILTER (WHERE rating IS NULL) / COUNT(*) AS pct_null_rating,
  COUNT(*) AS records
FROM scraped_records
WHERE created_at > NOW() - INTERVAL '12 weeks'
GROUP BY 1
ORDER BY 1;

-- Step changes = silent break
-- Example: pct_null_rating jumps from 8% to 45% in week of 2026-04-15 → selector broke that week
```

**Enum-like field drift:**

```sql
-- New values appearing in 'availability' column
SELECT availability, MIN(created_at) AS first_seen, COUNT(*)
FROM scraped_records
GROUP BY availability
ORDER BY first_seen DESC
LIMIT 20;
-- New value with recent first_seen + few records = parsing noise OR new source state worth handling
```

**Numeric distribution drift:**

```sql
SELECT
  DATE_TRUNC('week', created_at) AS week,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY price) AS median_price,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY price) AS p95_price,
  COUNT(*) AS records
FROM scraped_records
WHERE price IS NOT NULL AND created_at > NOW() - INTERVAL '12 weeks'
GROUP BY 1 ORDER BY 1;
-- Step change in median = currency parsing broke, OR commission added, OR market shift
```

**Pass criterion:** no >2× step change in null rate / median / unique-value count without a corresponding code change.

**Finding template (C3 fail):**
```
[SEV-4][S][consistency] `<field>` schema drift detected: <step change description>
  Trend data: <week-by-week table>
  Suspected cause: <site change on YYYY-MM-DD aligns with drift>
  Action: confirm via fixture comparison; restore selector with fallback; backfill if data is recoverable
```

### C4 — Cross-page consistency

**Question:** When the same entity appears on multiple page types, do extracted values agree?

```sql
-- Same product on category page vs. detail page
WITH paired AS (
  SELECT
    url_normalized,
    MAX(price) FILTER (WHERE source_page = 'category') AS price_category,
    MAX(price) FILTER (WHERE source_page = 'detail') AS price_detail,
    MAX(name) FILTER (WHERE source_page = 'category') AS name_category,
    MAX(name) FILTER (WHERE source_page = 'detail') AS name_detail
  FROM scraped_records
  WHERE created_at > NOW() - INTERVAL '7 days'
  GROUP BY url_normalized
  HAVING COUNT(DISTINCT source_page) > 1
)
SELECT
  COUNT(*) FILTER (WHERE name_category <> name_detail) AS name_disagrees,
  COUNT(*) FILTER (WHERE ABS(price_category - price_detail) > 0.01) AS price_disagrees,
  COUNT(*) AS total_paired
FROM paired;
```

**Pass criterion:** ≤5% disagreement.

**Finding template (C4 fail):**
```
[SEV-3][M][consistency] Cross-page disagreement: X% of `<field>` differ between category and detail
  Likely cause: category page shows abbreviated/cached value, detail has fresh
  Action: pick one as canonical (usually detail), document; OR enrich category records from detail
```

### C5 — Cross-source agreement *(when multiple sources scrape similar data)*

**Question:** Does source A's price ≈ source B's price for the same item?

```sql
WITH paired AS (
  SELECT
    product_key,
    AVG(price) FILTER (WHERE source = 'source-a') AS a_price,
    AVG(price) FILTER (WHERE source = 'source-b') AS b_price
  FROM scraped_records
  WHERE created_at > NOW() - INTERVAL '7 days' AND price IS NOT NULL
  GROUP BY product_key
  HAVING COUNT(DISTINCT source) > 1
)
SELECT
  100.0 * COUNT(*) FILTER (WHERE ABS(a_price - b_price) / NULLIF(a_price, 0) > 0.5) / COUNT(*) AS pct_diverge_50
FROM paired;
```

**Pass criterion:** material disagreements (>50% drift) ≤10%. Some drift is expected (different markets, different units, different inventory).

**Finding template (C5 fail):**
```
[SEV-3][M][consistency] Cross-source price disagreement >50% on X% of paired records
  Possible causes:
    - one source includes tax and the other excludes it
    - currency conversion bug on one side
    - one source returns bulk/case price, the other unit price
  Action: sample 20 paired records, inspect manually, identify root cause
```

---

## How to use these tests in practice

1. During Phase 1 (gather artifacts), pull the recent dataset / DB samples.
2. During Phase 2 (walk dimensions), run Q1–Q6 then C1–C5.
3. Failures become candidate findings with the template above.
4. Cross-reference the fix-doctrine reference (links provided per finding).

Don't run all tests on every audit — pick the ones most likely to surface bugs given the scraper's known issues. Q1, Q2, C2, C3 are the highest-ROI defaults.
