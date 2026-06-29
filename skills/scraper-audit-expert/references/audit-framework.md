# Audit Framework — 8 Dimensions

The full per-dimension checklist. Each dimension has:
- **What it answers** (1 sentence)
- **Artifacts needed** (what to gather)
- **Checklist** (concrete items, each producing a candidate finding)
- **Severity criteria** (how to rate findings in this dimension)
- **Cross-references** (where the fix doctrine lives)

This file is walked in order during Phase 2 of the audit. Skipping a dimension = unfinished audit.

---

## Phase 1 — Artifacts (gathered before Phase 2 walk)

### Common to both runtimes

- [ ] Full source tree (`find . -type f -name '*.ts' -o -name '*.py' -o -name '*.json' | grep -v node_modules`)
- [ ] `package.json` / `requirements.txt` / `pyproject.toml`
- [ ] `README.md` and any project/session notes
- [ ] Sample output ≥ 1k records, ideally 10k+ for statistical signal
- [ ] Known issues / user reports (verbatim, with reproduction steps if any)
- [ ] Git log of the last 90 days (`git log --oneline --since="90 days ago"`)
- [ ] Test fixtures (`ls tests/fixtures/` — how many, scrubbed?)

### Apify Actor

```bash
# Public-facing metadata + source files exposed via API
apify actors info <user>/<slug> --json > /tmp/audit-meta.json
jq '{stats, latestVersion: .versions[0].versionNumber, sourceFiles: [.versions[0].sourceFiles[].name]}' /tmp/audit-meta.json

# Recent runs (status, duration, items, charges)
apify runs ls --actor=<user>/<slug> --limit=100 --json > /tmp/audit-runs.json

# Success rate over last 100 runs
jq '[.[] | .status] | group_by(.) | map({status: .[0], count: length})' /tmp/audit-runs.json

# Mean duration + item count for SUCCEEDED runs
jq '[.[] | select(.status == "SUCCEEDED") | {duration: ((.finishedAt | fromdateiso8601) - (.startedAt | fromdateiso8601)), items: .stats.outputItems}] | {meanDuration: (map(.duration) | add / length), meanItems: (map(.items) | add / length)}' /tmp/audit-runs.json

# Sample dataset items (pick a recent SUCCEEDED run)
RUN_ID=$(jq -r '[.[] | select(.status == "SUCCEEDED")][0].id' /tmp/audit-runs.json)
apify dataset get <datasetId> --limit=1000 --json > /tmp/audit-dataset.json
```

### Standalone (Express/PM2 service)

```bash
# Process health + memory + restarts
pm2 list
pm2 show <process-name> | grep -E 'memory|restarts|status|uptime'
pm2 prettylist | jq '.[] | select(.name == "<process-name>") | {memory: .monit.memory, cpu: .monit.cpu, restarts: .pm2_env.restart_time}'

# Error pattern aggregation (last 7 days)
pm2 logs <process-name> --err --lines 10000 --nostream > /tmp/audit-errors.log
grep -iE 'error|fail|timeout|cascade|429|403|500' /tmp/audit-errors.log | sed -E 's/[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9:.]+Z?//g; s/[0-9a-f-]{36}//g' | sort | uniq -c | sort -rn | head -20

# Postgres: count + null rates per field per source
psql -d <db> -c "\d+ <main_table>"   # column inventory
psql -d <db> -c "SELECT source, COUNT(*) FROM scraped_records WHERE created_at > NOW() - INTERVAL '7 days' GROUP BY source;"
psql -d <db> -c "SELECT 100.0 * COUNT(*) FILTER (WHERE price IS NULL) / NULLIF(COUNT(*), 0) AS pct_null_price FROM scraped_records WHERE created_at > NOW() - INTERVAL '7 days';"

# Redis cache health
redis-cli INFO stats | grep -E 'keyspace_hits|keyspace_misses|expired_keys|evicted_keys'
redis-cli DBSIZE
redis-cli --scan --pattern 'cache:*' | head -20   # naming convention check

# Custom in-app metrics if present (an admin stats endpoint pattern)
curl -s http://localhost:<port>/api/admin/scrape-stats 2>/dev/null | jq .
```

---

## Dimension 1: Coverage

**What it answers:** What data is the scraper *actually* extracting vs. what's available on the target source?

**Why it matters:** Untapped fields are invisible by definition. They never produce a "bug" — just missed value. Most audits skip this dimension because finding gaps requires looking *outside* the scraper.

### Checklist

- [ ] Pick 10–20 representative target pages (popular + long-tail + edge cases like out-of-stock, product variants)
- [ ] For each: open DevTools → **view source AND rendered DOM** AND **`<script type="application/ld+json">`** blocks AND microdata (`itemprop` attributes)
- [ ] Build a "source field inventory" — every field present on the source
- [ ] List every field the scraper extracts (from TS types / Postgres schema / dataset_schema.json)
- [ ] **Diff**: source ∖ extracted = **coverage gaps**
- [ ] DevTools Network panel: any `/api/...` JSON endpoint with richer data? See your hidden-APIs doctrine
- [ ] *(if a managed API is in the pipeline)* Log the raw response before transformation; diff against persisted record
- [ ] Long-tail probe: out-of-stock, product variants, items without a current price, non-USD currencies, non-ASCII names, RTL text — does the scraper extract correctly?
- [ ] Pagination coverage: does the scraper actually traverse all pages, or stop early on rate limit?
- [ ] Entity-type coverage: is the scraper limited to one entity type (products) when the target has more (reviews, sellers, related products)?

Full systematic methodology: **`coverage-methodology.md`**.

### Severity criteria (Dim 1)

- **🔴 SEV-5** Coverage gap on a field that drives business decisions (e.g. `price`, `availability` missing → frontend broken)
- **🔴 SEV-4** Coverage gap on a routinely-displayed field affecting >10% of records
- **🟡 SEV-3** Untapped field that has value but isn't currently displayed
- **🟡 SEV-2** Long-tail entity types not extracted (e.g. reviews, related products)
- **🟢 SEV-1** Nice-to-have data exposed via JSON-LD but cosmetic

### Cross-references (for fixes)

- New selector for an existing source: your selector-strategy doctrine
- Hidden API discovery to replace HTML scraping: your hidden-APIs doctrine
- Adding pagination handlers: a Crawlee-template reference § routes pattern
- AI-extraction for unstructured fields: an AI-extraction reference

---

## Dimension 2: Quality

**What it answers:** How accurate, complete, conformant, and normalized is the data the scraper *does* produce?

**Why it matters:** Bad data masquerading as good data is the #1 source of downstream bugs.

### Checklist

- [ ] **Completeness per field** — query null rate by field over recent records (last 7 days)
- [ ] **Group nulls by source/locale** — a field 2% null overall but 40% null for UK = localization bug, not noise floor
- [ ] **Accuracy sample** — 100 records → manually verify each field vs. live source page → compute per-field error rate (group by error type: missing / wrong / stale / format)
- [ ] **Schema conformance** — run all records through declared types (Zod, JSON Schema, Pydantic). Count violations
- [ ] **Normalization checks:**
  - Currency: ISO 4217 codes only, no `$ £ €` raw symbols
  - Prices: numeric type, consistent precision (2 decimals), no thousand-separators in the value
  - Dates: ISO 8601 UTC, never "2 days ago" or locale strings
  - Strings: trimmed, NFC-normalized, no HTML entities (`&amp;amp;`)
  - Booleans: actual booleans, not `"yes"/"no"/1/0` mixed
  - URLs: absolute (no relative paths persisted), `https://` not `http://` if available
- [ ] **PII / safety** — no auth cookies / tokens / session IDs leaked into records
- [ ] **Junk records** — empty objects, error pages parsed as valid records, captcha HTML extracted as content

Full concrete test queries: **`quality-and-consistency-tests.md`** § "Quality tests".

### Severity criteria (Dim 2)

- **🔴 SEV-5** Wrong data on a critical field (e.g. price off by 10x) affecting >5% of records OR any leaked PII
- **🔴 SEV-4** Null rate >10% on a non-nullable field, or schema violations on >5% of records
- **🟡 SEV-3** Inconsistent normalization (mixed currency symbols, dates in two formats)
- **🟡 SEV-2** Untrimmed strings, HTML entities, minor formatting issues
- **🟢 SEV-1** Cosmetic (extra whitespace, capitalization inconsistency)

### Cross-references

- Selector with fallbacks: your selector-strategy doctrine
- Zod validation at extraction boundary: a Crawlee-template reference § extractors
- AI-extraction when selectors can't deliver: an AI-extraction reference § reliability patterns

---

## Dimension 3: Consistency

**What it answers:** Same input → same output across runs? Schema stable over time? Dedup correct?

**Why it matters:** Inconsistent data corrupts downstream analytics silently.

### Checklist

- [ ] **Cross-run stability** — pick 20 stable URLs, scrape today and again in 24h. Diff. Fields that change for stable products (price OK, name NOT OK) flag a parser bug
- [ ] **Cross-page consistency** — same entity on category page vs. detail page: do extracted values agree?
- [ ] **Dedup correctness** — query for `(natural_key)` group counts in Postgres / dataset. Expected duplicates = 0 (or known-merged)
- [ ] **Schema drift detection** — sample records grouped by scrape week. For each field, plot null rate / value distribution / unique count. Step changes = silent break
- [ ] **Enum stability** — for fields with bounded values (currency, country, availability), set-diff this week's distinct values vs. last month's. New values = real expansion OR parsing noise
- [ ] **Numeric distribution stability** — mean/median by week for `price`, `rating`. Step changes flag parser bugs (e.g. currency suddenly being concatenated into price)
- [ ] **Idempotency** — does the same input twice produce two records or one? For on-demand scrapers, this is critical (race conditions, cache stampedes)
- [ ] **Cross-source agreement** *(when multiple sources scrape similar data)* — source A price ≈ source B price within ±50%? Material disagreements flag one source as wrong

Full concrete queries: **`quality-and-consistency-tests.md`** § "Consistency tests".

### Severity criteria (Dim 3)

- **🔴 SEV-4** Sudden step change in null rate / value distribution = silent parser break
- **🔴 SEV-4** Duplicates >5% with no documented merge rule
- **🟡 SEV-3** Cross-page disagreements > 10%
- **🟡 SEV-3** Schema drift in enum-like fields (new values appearing)
- **🟢 SEV-2** Minor cross-source price drift expected to track between sources

### Cross-references

- Idempotent extraction patterns: a Crawlee-template reference
- Cache invalidation correctness: your scraping-patterns doctrine § "KV Store cache pattern"

---

## Dimension 4: Resilience

**What it answers:** What happens under site changes, rate limits, network failures, partial DOM, anti-bot escalation?

**Why it matters:** A scraper that works on the happy path but cascades on the first 403 has zero production value.

### Checklist

- [ ] **Selector fallbacks** — every critical field has `selector1, selector2, selector3` (multi-pattern). Greppable: `\$\(['"][^,]+['"]\)` finds single-pattern (fragile)
- [ ] **Graceful exit** *(Apify)* — `Actor.fail()` is reserved for infrastructure errors only. Business-logic errors push to dataset + `Actor.exit()` SUCCEEDED. Grep for `Actor.fail(`
- [ ] **No process.exit()** anywhere; only `crawler.autoscaledPool?.abort()` for mid-run abort
- [ ] **Anti-bot escalation discipline** — proxy starts at DATACENTER, escalates to RESIDENTIAL on observed blocks, not pre-emptively. Stealth browser only on confirmed bot-detection
- [ ] **Retry strategy** — `retryOnBlocked: true` for Crawlee, `withRetry` wrapper for raw HTTP. Max retries 3-5, exponential backoff
- [ ] **Timeout discipline** — `requestHandlerTimeoutSecs` explicit (60s HTTP, 120s browser). For standalone managed-API: per-call timeout < total run budget / 4
- [ ] **Cascade containment** — one slow upstream blocks N workers? Use semaphore (p-limit) not unbounded concurrency
- [ ] **Empty-page handling** — `cards.length === 0` pushes EXTRACTION_FAILED record + continues (does NOT throw)
- [ ] **Captcha / anti-bot interstitial detection** — a `hasUsefulData(parsed)` style guard. If page returns 200 but content is a CAPTCHA, the scraper must detect this BEFORE pushing
- [ ] **Charge AFTER success only** *(Apify PPE)* — never in catch blocks, never in retry loops; see Dim 8 / cross-ref to the PPE audit framework
- [ ] **PII redaction in error records** — `pushData({ error: true, input: redactPII(input) })` not raw input

### Severity criteria (Dim 4)

- **🔴 SEV-5** `Actor.fail()` called for business errors → tanked success rate (e.g. 87% observed when target is 95%+)
- **🔴 SEV-5** Cascade containment missing → one slow upstream takes down entire run
- **🔴 SEV-4** Single-selector fields with no fallback on critical extraction paths
- **🔴 SEV-4** Captcha/interstitial returned as parsed data
- **🟡 SEV-3** Generic retry without backoff or max-retries cap
- **🟡 SEV-3** Anti-bot pre-emptively escalated (RESIDENTIAL when DATACENTER would work)
- **🟢 SEV-2** Timeout values inherited from defaults (unset explicitly)

### Cross-references

- Graceful exit doctrine: your error-handling / PPE doctrine
- Anti-bot escalation tactics: your anti-bot-strategies doctrine
- Tool ladder (when to escalate browser vs. managed API): your tool-ladder doctrine
- `eventChargeLimitReached` handling: a PPE-implementation doctrine

---

## Dimension 5: Performance & Cost

**What it answers:** Is the scraper throughput/latency/cost where it should be?

**Why it matters:** A correct scraper that costs $0.50/record when the PPE price is $0.001/record is bankrupt.

### Checklist

- [ ] **$/successful-record** — total cost (Apify compute + managed-API spend) / records pushed = unit cost. Compare against PPE event price (Apify) or business value (standalone)
- [ ] **Cache hit rate** — Redis `keyspace_hits / (keyspace_hits + keyspace_misses)`. Target ≥80% for hot data. Apify KV cache: instrument and log hits
- [ ] **Concurrency tuning** — `maxConcurrency` / `maxRequestsPerMinute` set explicitly? At ~80% of target rate limit?
- [ ] **Memory tier match** — Cheerio at 256–512 MB, Playwright at 1–2 GB, stealth browser at 2–4 GB. Apify Console → run memory chart should plateau at 60–80% of cap
- [ ] **Cold start frequency** *(Apify Standby / MCP)* — see your MCP cold-start mitigations
- [ ] **Latency decomposition** *(standalone on-demand)* — measure: frontend → Express → cache check → external API → parse → DB write → response. Usually external API = 80% of time
- [ ] **Provider waste** — managed APIs charged per request: are there retries hitting paid provider for transient failures that should retry locally first?
- [ ] **N+1 patterns** — extractor calls Postgres per item? Should batch
- [ ] **Connection pool sizing** — pg pool default 10 is often wrong for high-concurrency Express
- [ ] **Sequential where parallel is safe** — e.g. fetching two independent sources in series instead of `Promise.allSettled`

### Severity criteria (Dim 5)

- **🔴 SEV-5** Cost-per-record > PPE event price → bankrupt unit economics
- **🔴 SEV-4** Cache hit rate < 40% on hot endpoints when 80%+ is achievable
- **🟡 SEV-3** Memory tier wasteful (Playwright at 4 GB when 1 GB works)
- **🟡 SEV-3** Concurrency hardcoded / unset → 429 cascades OR under-utilization
- **🟢 SEV-2** Sequential operations that could be `Promise.all`
- **🟢 SEV-2** Provider not yet swapped for cheaper alternative

### Cross-references

- Memory tier table: your scraping-patterns doctrine § "Memory tiers and cost"
- Managed-API cost comparison: your managed-APIs doctrine
- KV cache pattern: your scraping-patterns doctrine § "KV Store cache pattern"
- Concurrency / `maxRequestsPerMinute` rule: a Crawlee-template reference § main.ts

---

## Dimension 6: Observability

**What it answers:** Can a single failed record be debugged? Are operators alerted when the scraper degrades?

**Why it matters:** A scraper that breaks silently in production for 3 weeks is worse than one that crashes loudly on day 1.

### Checklist

- [ ] **Structured logs** — JSON output, not free-form text. Searchable fields: `level`, `source`, `target_url`, `record_id`, `latency_ms`, `error_type`
- [ ] **No PII in logs** — auth tokens, session cookies, raw user input redacted
- [ ] **Metrics emission** — counters for: requests by source, errors by type, charge events fired (Apify), cache hit rate. Histograms for latency p50/p95/p99
- [ ] **Alerting** — what wakes up oncall? At minimum: success rate drop > 10% in 1h, error rate > 5%, cost spike > 50%, charge events stop firing
- [ ] **Runbooks** — when an alert fires, what does the responder do? Document
- [ ] **Status messages** *(Apify)* — `Actor.setStatusMessage(..., { isStatusMessageTerminal: true })` on every exit path. Visible in Console
- [ ] **Error records pushed to dataset/DB** — not just logged. Errors must be queryable and aggregable
- [ ] **Charge audit trail** *(Apify PPE)* — log every `Actor.charge` call with eventName + count + ChargeResult so refund disputes are debuggable
- [ ] **Counter instrumentation** — per-provider request count, exposed via admin endpoint
- [ ] **Trace IDs** — each scrape job gets a trace ID propagated through logs, allowing cross-service debugging

### Severity criteria (Dim 6)

- **🔴 SEV-4** No alerting on success rate / cost → scraper degrades for weeks unnoticed
- **🔴 SEV-4** Errors swallowed in catch blocks, no structured error records
- **🟡 SEV-3** Free-form log text (not JSON), grep-only debugging
- **🟡 SEV-3** No latency / cost metrics — optimization is impossible to verify
- **🟡 SEV-2** Status messages missing or generic
- **🟢 SEV-1** Minor logging improvements

### Cross-references

- Structured logging with `apify` SDK `log` namespace: `apify` package docs
- Status message templates: your error-handling / PPE doctrine § "Status messages on the error path"

---

## Dimension 7: Architecture

**What it answers:** Is the code organized for change? Tested? Type-safe? Secrets-clean?

**Why it matters:** Audits surface architectural debt that compounds. Findings here prevent the next audit from being as long.

### Checklist

- [ ] **Separation of concerns** — `main.ts` thin orchestration; `routes.ts` per-label handlers; `extractors/*.ts` pure functions taking `($card, baseUrl)` and returning a typed object
- [ ] **Pure extractors** — testable in isolation against fixtures. No SDK calls inside extractor functions
- [ ] **Type safety** — Zod / JSON Schema / Pydantic at extraction boundaries. No `as any` casts in production code. Grep: `as any`
- [ ] **Test fixtures present** — `tests/fixtures/*.html` (scrubbed of PII), captured from real runs, committed to git
- [ ] **Test coverage** — at minimum one extractor unit test per critical field. Missing-field tests verify nulls
- [ ] **Mocking discipline** — `Actor.charge()`, `Actor.pushData()`, `Actor.openKeyValueStore()` mocked in tests. No live network calls
- [ ] **Secrets hygiene** — `.env`, `secrets/`, credentials in `.gitignore` and `.dockerignore`. No tokens in `actor.json` / `package.json` / source files
- [ ] **Apify source-files hygiene** *(Apify only)* — after `apify push`: `apify actors info --json | jq '.versions[0].sourceFiles[].name'` shows ONLY public files (no internal docs / business context). `isSourceCodeHidden: true` set
- [ ] **Dependency hygiene** — `npm audit` / `pip-audit` for known CVEs. Versions pinned (no `^x.y.z` for direct deps that have breaking changes)
- [ ] **Dead code** — exploratory `test-*.mjs` files cleaned up before push (fine for dev, not for a published Actor)
- [ ] **Documentation freshness** — README accurately describes current input/output. Project notes reflect current architecture, not last quarter

### Severity criteria (Dim 7)

- **🔴 SEV-5** Tokens / secrets leaked in `sourceFiles[]` (Apify) or committed to git (standalone)
- **🔴 SEV-5** Internal business docs (private notes, audit reports) leaked via `sourceFiles[]`
- **🔴 SEV-4** No fixture tests → bugs go to production
- **🟡 SEV-3** `as any` casts in production code, weak type safety at boundaries
- **🟡 SEV-3** Documentation drift > 90 days
- **🟢 SEV-2** Code organization could be improved (single 800-line file etc.)

### Cross-references

- Source-files hygiene: the Apify push / source-files exposure note (the API exposes `versions[0].sourceFiles[]`; the `isSourceCodeHidden` flag only closes the Console UI, not the API)
- Project structure: a Crawlee-template reference § "Project structure"
- Testing fixtures: your testing-fixtures doctrine

---

## Dimension 8: Monetization *(Apify only)*

**DELEGATE.** Do not duplicate a dedicated PPE/pricing audit framework here.

The monetization audit covers:
- Dim 0: Sunset/legacy status
- Dim 1: Pricing model fit
- Dim 2: Charge correctness (`Actor.charge` placement, `eventChargeLimitReached`)
- Dim 3: Exit code discipline (overlap with this skill's Dim 4 — cite the PPE audit as authoritative)
- Dim 4: Event taxonomy
- Dim 5: Pricing level
- Dim 6: Transparency

In the scraper audit report, the **Monetization** section should be a **summary** of findings from running the PPE audit separately, or a one-liner saying "no monetization findings — see the PPE/pricing audit dated <date> for the full pass".

---

## Cross-cutting checks (apply across all dimensions)

These are not their own dimension but show up everywhere:

- [ ] **Affiliate-link compliance** — if affiliate/referral parameters are part of your funnel, all relevant `apify.com/...` URLs in README / Store description / public docs carry the agreed referral parameter consistently
- [ ] **Language hygiene** — internal docs in one canonical language; UI / customer-facing copy in its target language
- [ ] **Git remote check** — `git remote get-url origin` confirms the intended remote visibility (private vs. public) before any push
- [ ] **Compliance / ToS** — robots.txt respected? PII handling consistent with GDPR? Rate limits respected by configuration?

---

## Quantifying impact (for every 🔴 finding)

A finding without quantified impact is noise. Three methods:

### Method 1 — Apify Console data

```bash
# Total runs affected by issue X (last 30 days)
apify runs ls --actor=<user>/<slug> --limit=1000 --json | \
  jq '[.[] | select(.startedAt > (now - 30*86400 | strftime("%Y-%m-%dT%H:%M:%S.000Z")))] | length'

# Revenue at risk = affected runs × mean items × event price
# Example: 2000 runs × mean 12 items × $0.005 × 15% affected = $18/month
```

### Method 2 — Postgres / dataset queries (standalone)

```sql
-- % of records with the bug pattern over last 30 days
SELECT
  100.0 * COUNT(*) FILTER (WHERE price > 1000 OR price < 0) / NULLIF(COUNT(*), 0) AS pct_bad_price
FROM scraped_records
WHERE created_at > NOW() - INTERVAL '30 days';
```

### Method 3 — Sample-based extrapolation

When direct measurement is impossible, sample N=100 records, count instances, extrapolate. Always state the sample size and confidence interval explicitly: *"7/100 records (95% CI: 3-14%) had wrong currency."*

**Never** quote impact without showing the math.

---

## Sequencing the fixes

Three constraints:

1. **Apify 14-day rule** — these changes go live **immediately** when pushed: price *decreases*, event *removals*, code-only changes, README/docs updates, discount opt-ins, "PPE + usage" toggle off. These take **14 days** to apply: price *increases*, *new* events, pricing-model switches (PPE↔Pay-per-Usage), schema-breaking changes to declared events. **Only ONE delayed change per Actor per month** is allowed by the platform. Sequence: bundle all immediate fixes in a single push; reserve the slot for the highest-ROI delayed change. Full doctrine in a dedicated PPE/pricing audit framework.
2. **Coverage before quality** — adding a missing field shouldn't conflict with re-extracting an existing one. Sequence coverage findings before quality findings on the same field.
3. **Observability before optimization** — never recommend a perf optimization you can't measure post-fix. If observability gaps exist (Dim 6), fix them first.

Output of Phase 4 = two lists:

**Quick wins** (effort S, SEV 3-4): top 5-10. The "fix this week" list. Each ≤ 1 day of work.

**Strategic** (effort M/L, SEV ≥ 3): top 5-10. The "fix this quarter" list. Each requires planning, design, or architectural buy-in.

---

## When to defer or refuse an audit

Hard defer:
- **Scraper is < 7 days old** — no production data to audit. Use a pre-publish checklist instead.
- **Scraper is < 100 records in production** — sample size too small for statistical signal. Audit later.
- **Code is in active rewrite** — auditing a moving target is wasted effort. Wait for stabilization.
- **No observability at all** (no logs, no metrics, no error records) — first deliverable is "add observability"; full audit deferred.

In these cases, deliver a 1-page "pre-audit findings" memo: what to add before re-requesting an audit.

### Partial-artifact decision matrix

The line between "audit anyway with caveats" and "defer" when some artifacts are missing:

| Artifact available | Decision | Dimensions deferred to "Needs more data" |
|---|---|---|
| Source code + ≥1k records + logs + metrics | **Full audit, all 8 dimensions** | — |
| Source + records + logs, no metrics | **Audit with caveats** | Dim 5 perf items requiring instrumentation; Dim 6 observability findings labeled "audit-induced visibility gap" |
| Source + records, no recent logs | **Audit with caveats** | Dim 4 latent-bug discovery limited to grep candidates (can't confirm via log archaeology); Dim 6 fully deferred |
| Source + ≤100 records | **Coverage audit only** — defer full quality/consistency analysis until sample size grows. Run Coverage methodology + Architecture review. | Dim 2/3 limited to schema conformance |
| Source only, no production data | **Treat as pre-audit code review** — run latent-bug greps + a pre-publish checklist. Mark as "Code review, not audit." | Dim 1/2/3/5/6 all "no production data" |
| No source access, only Apify Console + sample dataset | **Run black-box audit** — coverage probe (Step 1, 2, 4 of `coverage-methodology.md`) + quality tests Q1/Q2/Q4/Q5/Q6 on dataset only. Findings tagged "black-box". | Dim 4/7 fully deferred (require code) |
| **Tool failure mid-audit** (CLI returns malformed JSON, missing dep like `jq`, auth/permission denied, rate-limited) | **Note the failure in the report's Methodology & scope section + Appendix A.1**, then proceed with degraded artifacts. Try workarounds (see below) before deferring a dimension. | Whichever dimension relied on the failed artifact, partially — quantification labeled "Inferred" or moved to Open Questions |

**Tool-failure workarounds** (apply in order before deferring a dimension):

| Failure | Workaround |
|---|---|
| `apify actors info --json` returns unparseable / truncated JSON | Fetch the same data via Apify REST API directly (`curl https://api.apify.com/v2/acts/<id>?token=$APIFY_TOKEN`) and pipe through `python3 -m json.tool` or `jq` with `--stream` flag for large payloads. OR scope down: `apify actors info <id> --json | head -c 1000000` for the smaller fields (stats, pricing). |
| `jq` not installed | Use `python3 -c "import sys, json; d=json.load(sys.stdin); print(...)"`. Document the inline Python in Appendix A.1 alongside the equivalent `jq` query for reproducibility. |
| Apify Console API auth fails | Run the audit with public-API-only data (the public actor-details surface most users see). Mark Dim 5/6 partial. |
| `pm2 logs` / Postgres unreachable | Note in scope; defer Dim 5/6 to "Needs more data" with the specific access needed as unblocker. |
| Rate-limited (Apify / managed API / database) | Pause, document the rate limit hit in Methodology, retry with backoff. If recurrent, switch to local-cached / pre-pulled artifacts where possible. |

**Important:** a tool failure does NOT mean "skip the dimension." It means "note the constraint, try a workaround, and label findings derived from incomplete artifacts with Confidence: Medium/Low or move them to Open Questions." Findings derived from working artifacts retain full Confidence.

When deferring a dimension, **always** list it in the report's "Open questions / Needs more data" section with the specific artifact needed to unblock.
