# Coverage Methodology — Five-Step Systematic Process

The hardest audit dimension. Most auditors only review what the scraper EXTRACTS; this methodology systematically identifies what the scraper COULD extract but isn't.

This file is the canonical procedure for Dimension 1 in `audit-framework.md`.

## Why coverage is hard

- **Untapped fields are invisible.** They don't produce errors. They don't show up in logs. They simply don't exist in the output.
- **Most scrapers are written against a single representative page** and never re-validated when the target adds new fields.
- **Schema.org markup often has 3–5× more fields than the visible DOM.** A scraper that reads `<span class="price">` misses `<meta itemprop="priceCurrency">`, `<meta itemprop="availability">`, `<meta itemprop="seller">`, etc.
- **Hidden APIs** often expose 10× more structured data than what's HTML-scraped.

## The five steps

### Step 1 — Build a source field inventory (manual)

Pick 10–20 representative target pages:
- 5 popular records (top-selling products, top-rated restaurants, etc.)
- 5 long-tail records (low-volume, edge cases)
- 5 known-difficult records (out-of-stock, product variants, items without a current price, multi-locale, RTL text, non-ASCII names)
- 5 random sample for variance

For each page:

```
[1] Open in browser
[2] View Source (Ctrl+U) — capture <script type="application/ld+json"> and microdata
[3] Inspect Element — capture itemprop attributes, data-* attributes
[4] DevTools Network panel filtered to XHR/Fetch — capture any /api/... JSON responses
[5] Disable JavaScript and reload — see what's server-rendered vs. client-injected
```

Build a spreadsheet:

| Field | Source page 1 | Source page 2 | ... | Source page 20 | Coverage rate |
|---|---|---|---|---|---|
| `name` | ✓ | ✓ | ... | ✓ | 20/20 |
| `price` | ✓ | ✓ | ... | ✗ (sold out) | 18/20 |
| `availability` | `InStock` | `InStock` | ... | `OutOfStock` | 20/20 |
| `seller_rating` | `4.5/5 (200 reviews)` | `–` | ... | ... | 12/20 |
| `variant` | `Large` | `Medium` | ... | `Small` | 20/20 |
| `category_hierarchy` | `Home > Outdoor > Tents` | ... | ... | ... | 16/20 |
| ... | ... | ... | ... | ... | ... |

**Critical:** include fields you find in JSON-LD, even if they're not visible. Schema.org `Product`, `Offer`, `Organization`, `Review` are the common heavy-data sources.

**Persist this spreadsheet as Appendix A.4 in the audit report** (per `audit-report-template.md` § Appendix A.4 — "Coverage probe — JSON-LD fields found vs. extracted"). It becomes the auditable evidence for every coverage finding.

### Step 2 — Hidden-API probe

Open DevTools Network panel:
1. Filter to **XHR/Fetch** only
2. Reload the target page
3. For each JSON response, click → preview the JSON tree
4. Look for endpoints with **richer data** than the HTML version

Common patterns:

| Site type | Hidden API pattern | What it typically exposes |
|---|---|---|
| E-commerce SPA | `/api/products/<id>`, `/_next/data/...` | Full product object with seller info, related products, inventory |
| GraphQL | `/graphql` | The frontend's actual query — often has fields not displayed |
| Embedded data | `<script>window.__INITIAL_STATE__ = {...}</script>` | Full hydration state |
| Search SPA | `/api/search?q=...` | Paginated results with fields the listing UI hides |

**If a hidden API exists, the entire scraping strategy is suspect.** Scraping HTML when the frontend already gets JSON is wasted work. See your hidden-APIs doctrine for the canonical pattern.

Document any hidden API found in the audit as `[coverage] Hidden JSON endpoint at <url> exposes <N> additional fields including <list>; current scraper reads HTML instead.`

### Step 3 — Managed-API raw diff *(applies when a managed API like ZenRows / Firecrawl / ScrapFly is in the pipeline)*

When the scraper uses a managed API (e.g. one provider for an anti-bot-protected source), the API returns more than what gets persisted. Check both:

**Raw response (what the managed API returns):**
- HTML-scraping providers: full HTML with the anti-bot challenge bypassed
- Firecrawl: markdown + raw HTML + screenshot + (optionally) `extract` JSON if schema is provided
- ScrapFly: parsed structured data when `extraction_template` is set

**Persisted record (what ends up in Postgres / dataset):**
- After the parser (e.g. a `parseHtmlPage` step) extracts a subset
- After validation (e.g. a `hasUsefulData` guard) potentially drops the whole record

**Diff procedure:**
1. Capture 20 raw responses (log them, write to disk during a controlled run)
2. Run the existing parser on each
3. List fields in raw response vs. fields in parsed output
4. **Anything in raw that's not in parsed = candidate untapped field**

In TypeScript pseudo-code:

```ts
// Add to the scraper for one audit run, then remove
import { writeFileSync } from 'fs';

const raw = await scrapeViaManagedApi(url, country);
writeFileSync(`audit-raw/${slugify(url)}.html`, raw);

const parsed = parseHtmlPage(raw);
writeFileSync(`audit-parsed/${slugify(url)}.json`, JSON.stringify(parsed, null, 2));
```

Then offline: for each raw file, grep for schema.org `itemprop` attributes and JSON-LD; compare against parsed output keys.

### Step 4 — Schema.org probe (fast)

For HTML-based scrapers, every `itemprop` is a candidate field. Run this on a single representative fixture:

```bash
# List all itemprop values present in the fixture
grep -oE 'itemprop="[^"]+"' tests/fixtures/target-page.html | sort -u

# List all itemtype values (entity types present)
grep -oE 'itemtype="[^"]+"' tests/fixtures/target-page.html | sort -u

# Extract all JSON-LD blocks
grep -oP '(?s)<script type="application/ld\+json">\K[^<]+(?=</script>)' tests/fixtures/target-page.html | jq .
```

Compare against `src/extractors/*.ts` — for each `itemprop` not consumed, the field is a coverage gap.

**Bonus:** check `itemref` attributes for cross-element references — these surface when a single entity has properties scattered across the DOM, easy to miss.

### Step 5 — Long-tail probe (find edge-case gaps)

Coverage often works for popular records and breaks on edges. Specifically test:

| Edge case | What to check |
|---|---|
| **Out-of-stock** | Does `price`, `availability`, `seller` extract correctly when stock = 0? |
| **Product variants** | When a product lists multiple variants (sizes, colors, editions): does the scraper extract them as one record (wrong) or N records (right)? |
| **No current price** | Items with "price on request" / pre-order / unavailable: does the scraper crash, push null, or correctly tag the record? |
| **Non-USD currencies** | EUR/GBP/CHF: does currency normalization work? Or do GBP prices end up unparsed (`"£42.50"` as string)? |
| **Non-ASCII names** | Accented characters (`Café` vs. `Cafe`): NFC vs. NFD unicode normalization? |
| **RTL text** | Arabic / Hebrew product descriptions: rendered correctly? Reversed? |
| **Long descriptions** | Does the scraper truncate at 255 chars? At what limit does data loss start? |
| **Multi-image products** | Does the scraper extract only the first image, or all? |
| **Reviews / ratings** | Are individual reviews scraped, or just aggregate score? |
| **Related products** | Is the "you might also like" panel scraped as a separate entity? |
| **Seller / merchant info** | Multi-merchant marketplaces: is each merchant's offer scraped? Their rating? Their country? |
| **Promotions / discounts** | Original price vs. sale price: both captured? Discount %? |
| **Stock quantity** | "Only 3 left in stock!" : numeric value captured? |
| **Estimated delivery** | Shipping promise text or date: captured? |

For each edge case the scraper handles wrong → `[coverage]` finding with the specific URL and the wrong/missing behavior.

## How to grade coverage findings

Severity based on **value of the field × % of records where extractable but not extracted**:

| Severity | Pattern |
|---|---|
| **🔴 SEV-5** | Critical business field (price, availability) missing — frontend / billing affected |
| **🔴 SEV-4** | Routinely-displayed field missing on >10% of records |
| **🟡 SEV-3** | Field has measurable value (drives search ranking, segmentation) but not currently surfaced |
| **🟡 SEV-2** | Long-tail entity type not extracted (reviews, related products, multi-merchant) |
| **🟢 SEV-1** | Nice-to-have field in JSON-LD but no immediate business case |

## Effort estimation

| Effort | Coverage gap shape |
|---|---|
| **S** (<1 day) | New selector for an existing source page; same parser, one new field |
| **M** (1-5 days) | New entity type extraction (e.g. reviews) requiring new route handler + dataset_schema update |
| **L** (>5 days) | Switching from HTML scraping to hidden-API consumption (full rewrite of the source layer) |

## Coverage finding template (for the report)

```markdown
### [coverage][SEV-N][Effort] <field-name or pattern> not extracted

**Source evidence:** <fixture file path or live URL>, <selector or JSON path>
  Example: `tests/fixtures/product-detail.html` line 142 has
  `<meta itemprop="aggregateRating" content="4.5">` — currently dropped.

**Persisted record evidence:** <sample record from dataset/Postgres>
  Example: `rating` field is absent in the schema; query
  `SELECT DISTINCT keys FROM scraped_records LIMIT 1` confirms.

**Impact:** <% of records where extractable> × <business value>
  Example: 78% of product detail pages expose ratings (16/20 sampled).
  Frontend would display this; currently shows "—".

**Fix direction:** <high-level — cross-ref the build doctrine for deep recipes>
  Add `aggregateRating` extraction to the detail parser. Update the
  product type + `dataset_schema.json`. Per your selector-strategy doctrine
  § "Schema.org markup", prefer `[itemprop]` selectors with fallback to a
  visual `.rating-value` class.
```

## When coverage is genuinely complete

If you ran steps 1–5 and found no gaps:
- Score Dim 1 = **5/5**
- Note in the report: "Coverage audit found no gaps. The scraper extracts all fields present in JSON-LD, microdata, and the rendered DOM across N=20 sample pages. Long-tail edge cases (out-of-stock, product variants, non-USD currencies) handled correctly."

This is rare. If you find no gaps in your first audit, recheck step 2 (hidden-API probe) and step 5 (long-tail probe) — those are usually where gaps hide.

## What this file does NOT cover

- **Pagination depth** is covered in `audit-framework.md` Dim 1 directly (not its own methodology — just "does it traverse all pages?").
- **Field accuracy** (extracted value ≠ source) is Dim 2 (Quality), not Coverage.
- **Schema drift detection** (a field that USED to be extracted but stopped) is Dim 3 (Consistency).
