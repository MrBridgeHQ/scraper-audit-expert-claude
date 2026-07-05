---
name: scraper-audit-expert
description: Use when auditing an existing scraper - Apify Actor or standalone (Express/PM2, FastAPI, cron, etc.) - to surface coverage gaps, data-quality issues, consistency bugs, resilience holes, performance/cost waste, observability gaps, and architectural debt. Triggers on "audit this scraper", "review my scraper", "what's wrong with X", "why does scraper Y fail", "is this scraper extracting all available data", "what's the data quality", "find bugs in scraper", schema drift detection, selector rot diagnosis, scrape cost-per-record analysis, success-rate diagnosis. Distinct from PPE/pricing audits (cross-referenced, not duplicated), scraping doctrine for fixes, and scraper/MCP build skills (build, not audit). Produces a structured audit report with severity-classified findings and prioritized fixes.
---

# Scraper Audit Expert

Senior-level audit of an existing scraper to identify what's wrong, what's missing, and what could be optimized. The output is a structured report (executive summary + scorecard + prioritized findings + fixes) - not a stream of "consider X".

This skill is opinionated about two things: **(1) all dimensions matter equally until proven otherwise** - don't skip coverage because "the data looks fine", don't skip observability because "it works". **(2) findings without quantified impact are noise** - every 🔴 finding must include "% of records affected" or "$ at risk" or "users blocked".

## When to use

- Existing scraper is "working" but you suspect data quality issues.
- Success rate dropped without code change → selectors rotted, site changed, or anti-bot escalated.
- Costs creeping up (managed-API or Apify compute) → cache hit rate or concurrency mistuned.
- New maintainer needs to understand what the scraper actually does vs. claims.
- Pre-launch dual review: pair with a pre-publish checklist before publishing.
- Post-incident: a bug was found; audit nearby surface area for similar issues.

## When NOT to use

- Scraper is **fully broken** (success rate < 30%) - that's triage, not audit. Fix the obvious failure first, audit after.
- Scraper is **being designed** (not yet implemented) - use a scraper/MCP build skill for build doctrine.
- Question is **purely about pricing/monetization** - use a dedicated PPE/pricing audit framework directly.
- Question is **purely about anti-bot** ("how do I get past Cloudflare?") - use an anti-bot strategies reference directly.

## Two runtimes the audit must distinguish

The 8 dimensions below apply to both, but the **artifacts to gather** and **commands to run** differ. Tagged as **[Apify]** and **[Standalone]** where relevant.

| Runtime | Examples | Observability surface |
|---|---|---|
| **Apify Actor** | A published Actor (e.g. an e-commerce or maps scraper), or any Actor in your portfolio | Apify Console: Runs / Insights / Billing tabs + Apify API (`apify actors info`, `apify runs ls`, `apify dataset get`) |
| **Standalone** | An Express+TS service on a VPS under PM2, other PM2 daemons, FastAPI services, cron jobs | Logs (`pm2 logs`, `journalctl`), Postgres queries, Redis stats, custom in-app counters, optional APM |

The audit framework is **runtime-agnostic at the dimension level**; the **gather artifacts** step branches on runtime.

## The 8 dimensions

| # | Dimension | What it answers |
|---|---|---|
| 1 | **Coverage** | What data the scraper *could* extract vs. what it actually does. Untapped fields, missing pagination, missing entity types. |
| 2 | **Quality** | Completeness (null rates), accuracy (vs. ground truth), schema conformance, normalization (currency, dates, encoding). |
| 3 | **Consistency** | Cross-run stability, cross-page agreement, dedup correctness, schema drift over time. |
| 4 | **Resilience** | Selector fallbacks, graceful exit, anti-bot escalation, timeout handling, cascade containment. |
| 5 | **Performance & Cost** | Throughput, latency p50/p95/p99, $/successful-record, cache hit rate, concurrency tuning. |
| 6 | **Observability** | Logs structured? Metrics emitted? Alerting on degradation? Can a single failed record be debugged? |
| 7 | **Architecture** | Code organization, testability, type safety, fixture coverage, secrets hygiene. |
| 8 | **Monetization** *(Apify only)* | **Delegate to a dedicated PPE/pricing audit framework** - do not re-audit here. |

Full per-dimension checklist with concrete commands: **`references/audit-framework.md`**.

## Severity classification

Two-axis classification + numeric severity.

**Severity (1–5):**
- **5 🔴 Critical** - data loss, billing error, security/PII leak, scraper materially broken (>30% of records affected or business-critical field wrong).
- **4 🔴 Critical** - silent quality issue affecting >10% of records, OR latent bug that becomes critical at scale.
- **3 🟡 Important** - quality issue <10%, latent reliability risk, observability gap that blinds operators.
- **2 🟡 Important** - inefficiency, tech debt, missed optimization with measurable cost.
- **1 🟢 Nice-to-have** - style, naming, documentation polish.

**Effort (S/M/L):** S = <1 day, M = 1-5 days, L = >5 days / architectural change.

**Category tag** - one of: `coverage`, `quality`, `consistency`, `resilience`, `perf`, `cost`, `obs`, `arch`, `monetization`, `compliance`.

**Finding format:** `[SEV-N][Effort][category] Title`. Example: `[SEV-4][S][quality] Currency parsing fails for GBP - 12% of UK records affected`.

**Prioritization:**
1. Anything **SEV-5** to the top regardless of effort.
2. Then sort by `severity / effort`.
3. Bump **+1 sev** for findings tied to active user complaints or recent incidents.
4. Findings where impact can't be quantified get **SEV-?** and go to a separate "Needs more data" section. **Never fake confidence on severity.**

## The audit workflow

Five phases. Skip a phase = audit drifts into a hot-take.

### Phase 1 - Gather artifacts (no code analysis yet)

Determine the runtime, then gather:

**Common to both runtimes:**
- Full source tree (read-only - fixtures + extractor + routes + main entry point + types/schemas)
- `package.json` / `requirements.txt` for dependency health
- `README.md` and any project/session notes
- Sample output: **≥ 1,000 records** for statistical signal, **≥ 10,000** preferred
- Known issue list / user-reported bugs (verbatim)

**Apify-specific:**
```bash
# Actor metadata + source files visible to the world
apify actors info <user>/<slug> --json | jq '{stats, latestVersion: .versions[0].versionNumber, sourceFiles: [.versions[0].sourceFiles[].name]}'

# Recent runs (last 100) with status + duration + item count
apify runs ls --actor=<user>/<slug> --limit=100 --json | jq '[.[] | {id, status, startedAt, finishedAt, stats: {compute: .stats.computeUnits, items: .stats.outputItems, charges: .stats.totalChargedAmountUsd}}]'

# Sample dataset items
apify dataset get <datasetId> --limit=100 --json

# Build hygiene
apify actors info <user>/<slug> --json | jq '.versions[0].sourceFiles[].name' | sort
```

**Standalone-specific:**
```bash
# Process health
pm2 list && pm2 logs <process-name> --lines 500 --nostream

# Recent error patterns (last 7 days)
pm2 logs <process-name> --err --lines 5000 --nostream | grep -iE 'error|fail|timeout|429|403|cascade' | sort | uniq -c | sort -rn | head -20

# Postgres: sample + null rate per field
psql -d <db> -c "SELECT field, COUNT(*) FILTER (WHERE field IS NULL) * 100.0 / COUNT(*) AS pct_null FROM <table> GROUP BY field;"

# Redis stats
redis-cli INFO stats | grep -E 'keyspace_hits|keyspace_misses|expired'

# Scraping provider usage (if an in-app stats endpoint exists)
curl -s http://localhost:<port>/api/admin/scrape-stats | jq .
```

Full artifact catalog: `references/audit-framework.md` § "Phase 1 artifacts".

### Phase 2 - Walk the 8 dimensions

In order: Coverage → Quality → Consistency → Resilience → Performance & Cost → Observability → Architecture → (Monetization, Apify only).

The order matters: **you cannot measure quality on a field you didn't know existed** (covers Coverage→Quality dependency), and **you cannot diagnose flakiness systematically without consistent samples** (Quality→Consistency).

For each dimension, walk the checklist in `references/audit-framework.md` and tag findings.

### Phase 3 - Quantify every 🔴 finding

For each Critical or Important finding:
- **Apify:** look at `apify runs ls` last 30d, compute % of runs/items affected, multiply by event price for revenue impact
- **Standalone:** SQL `COUNT(*)` queries on Postgres to compute affected rows
- **Both:** if quantification is impossible from available data, label `SEV-?` and add to "Needs more data" section

**Never fake a number.** "~12% of records" with a query attached >> "many records".

### Phase 4 - Sequence the fixes

Three constraints to respect:
1. **Apify 14-day rule** - price increases / event additions take 14 days; only ONE major change per Actor per month. See a PPE/pricing audit framework for the full doctrine.
2. **Coverage before quality fixes** - adding a missing field shouldn't conflict with re-extracting an existing one.
3. **Observability before optimization** - never recommend a perf optimization that you can't measure after the fact. Fix obs gaps first if blind.

Output two lists at the end of the report:
- **Quick wins** (SEV 3-4, effort S) - "fix this week"
- **Strategic** (SEV ≥ 3, effort M/L) - "fix this quarter"

### Phase 5 - Deliver the audit report

Use the template in **`references/audit-report-template.md`**. Length discipline: **8–15 pages total**. If you have > 20 findings, cut to top 20 and link the rest in an appendix.

## The audit deliverable structure

```
1. Executive summary (½ page) - top findings, overall health (color), effort-to-impact summary
2. Methodology & scope (½ page) - what was looked at, what wasn't, sample sizes, time windows
3. Snapshot table - runs, items, success rate, current state per Actor / per cron job
4. Findings 🔴 Critical - full Diagnostic / Impact / Fix blocks (cross-ref the relevant doctrine for deep fixes)
5. Findings 🟡 Important - same structure, condensed
6. Findings 🟢 Nice-to-have - bullet list
7. Dimension scorecard - 8 dimensions × 5-point scale + one-paragraph justification per dim
8. Quick wins list - sorted SEV/effort, the "fix this week" punch list
9. Strategic recommendations - the "fix this quarter" list
10. Open questions / Needs more data - non-negotiable section, never skip
11. Appendix - raw queries used, sample bad records, null-rate tables
```

Tone rules:
- **Direct, technical, no hedging on facts** but **explicit hedging on inferences**. "Field X has 12% null rate" (fact) vs. "this likely costs ~$Y/month" (inference, labeled).
- **No blame language** - findings are about the system, not the engineer who wrote it.
- **Propose, don't prescribe** - recommended direction (1-3 sentences), not a PR. Exception: trivial one-liner fixes go inline.

## Cross-references for fixes

The audit **identifies** problems and **cross-references** the right doctrine for the fix. Do not duplicate fix recipes in the audit report - link to the doctrine.

| Finding category | Where the fix doctrine lives |
|---|---|
| Selector rot, missing fallbacks, schema.org under-use | Your selector-strategy / scraping doctrine |
| Wrong tool (Cheerio for SPA, Playwright for static) | Your tool-ladder / scraping doctrine |
| Anti-bot mis-escalation, 403/429 cascade | Your anti-bot-strategies doctrine |
| `Actor.fail()` for business errors, no graceful exit | Your error-handling / PPE doctrine |
| Wrong `Actor.charge()` placement, missing `eventChargeLimitReached`, double-charge | A PPE-implementation / anti-patterns doctrine |
| PPE pricing model / price level | A dedicated PPE/pricing audit framework |
| Missing `requestHandlerTimeoutSecs`, wrong memory tier, no fixture tests | A scraper-build / Crawlee-template reference + pre-publish checklist |
| Apify Store README / pricing-section incoherence | An Actor content / coherence-audit doctrine |
| Hidden API not used (HTML scraping when JSON exists) | Your hidden-APIs doctrine |
| Managed-API choice (one provider vs. another) | Your managed-APIs comparison doctrine |

## Coverage discovery - the hardest dimension

This is where most audits fail: identifying **what the scraper could be extracting but isn't**. Most auditors only audit what's there. Full systematic methodology in **`references/coverage-methodology.md`**. Five-step process:

1. **Source field inventory** - Manual catalog of every visible field on 10–20 representative target pages, including JSON-LD/microdata (often richer than visible DOM).
2. **Hidden-API probe** - Open DevTools Network, filter to XHR/Fetch. If a `/api/...` endpoint returns JSON with more fields, **the entire scraping strategy is suspect**. See your hidden-APIs doctrine.
3. **Managed-API raw diff** *(applies when a managed API like ZenRows/Firecrawl/ScrapFly is in the pipeline)* - log the raw response before transformation. Diff against what the persisted record contains. Fields dropped between raw and persisted are often un-extracted by oversight.
4. **Schema.org probe** - `grep -oE 'itemprop="[^"]+"' <fixture.html> | sort -u` on a representative fixture. Every `itemprop` is a candidate field.
5. **Long-tail probe** - coverage works for popular records and breaks on edges. Specifically test: out-of-stock, product variants, items without a current price, non-USD currencies, non-ASCII names.

Coverage findings get prefixed with `[coverage]` and graded by:
- **Impact:** value of the field × % of records where it's extractable
- **Effort:** is it one new selector or a parser rewrite?

## Latent bug discovery - systematic patterns

The other dimension where baseline auditors rely on intuition. Full pattern catalog in **`references/latent-bug-patterns.md`**. Top 10 to grep for:

| Pattern | Grep | Why it's a bug |
|---|---|---|
| Empty catch | `catch\s*\{\s*\}` or `catch\s*\([^)]*\)\s*\{\s*\}` | Silent swallow |
| Optional chain on parsed data | `data\?\.\w+\?\.` | `??` makes bad data look good |
| `\|\| null` / `\|\| 0` defaults | `\|\|\s*(null\|0\|''\|""\|\[\])` | Hides extraction failures |
| `Promise.all` over agent calls | `Promise\.all\(\[.*runAgent` | One failure kills the batch - use `allSettled` |
| `Actor.fail` for business errors | `Actor\.fail\(` outside infra-error path | Tanks success rate |
| Hardcoded selectors no fallback | `\$\(['"][^,'"]+['"]\)` (no comma) | Brittle against A/B tests |
| Hardcoded URLs | `https?://[^/]+/.+` in source | Configuration drift |
| `as any` casts | `as any` | Type safety waived |
| Synchronous fs in hot path | `readFileSync\\|writeFileSync` | Event loop block |
| Cache write without invalidation | `setValue` without matching delete | Stale data forever |

Each grep produces candidate findings; the auditor confirms by reading context. **Greps surface candidates, not findings.**

## Common auditor mistakes

| Mistake | Why it's wrong |
|---|---|
| Reporting findings without quantified impact | "This might affect users" is noise. Pull the % from data or label SEV-?. |
| Auditing code without sampling output | Bugs hide in data, not always in code. Sample ≥ 1k records before opening the editor. |
| Skipping Coverage because "data looks complete" | Untapped fields are invisible by definition. Always run the JSON-LD probe. |
| Recommending fixes the audit can't measure post-hoc | If observability is broken, fix obs first, then optimize. |
| Re-auditing monetization findings already in the PPE audit scope | Cross-reference, don't duplicate. The PPE audit lives in a dedicated framework. |
| Treating Apify and standalone identically | Artifacts/commands differ. Tag findings with runtime. |
| Length > 20 findings | Cut to top 20, appendix the rest. Audit fatigue kills follow-through. |
| Fake severity confidence on inferred impact | `~$Y/month` without a query attached is a guess. Label "Inferred" or move to SEV-?. |
| Blaming engineers | Findings are about the system. Tone is non-negotiable. |
| Recommending fixes via copy-paste of pricing recipes | Cross-ref the canonical doctrine. Don't fork it. |

## Reference files in this skill

- `references/audit-framework.md` - Full 8-dimension checklist with commands, jq queries, SQL templates, severity criteria per dimension
- `references/coverage-methodology.md` - Five-step systematic coverage analysis (manual + tooling)
- `references/quality-and-consistency-tests.md` - Concrete tests for null rates, schema drift, dedup correctness, cross-source agreement
- `references/latent-bug-patterns.md` - 30+ grep patterns + why each is a bug + how to confirm
- `references/audit-report-template.md` - Full markdown template the auditor fills in

## What this skill is not

- **Not a replacement for a PPE/pricing audit** - the PPE audit is not duplicated here. Cross-reference for monetization findings.
- **Not a build skill** - for "how do I build a scraper", use a scraper-build skill. This skill audits existing code.
- **Not an anti-bot recipe** - for "how do I bypass Cloudflare", use an anti-bot strategies reference. This skill identifies anti-bot mis-configuration; the doctrine above fixes it.
- **Not a generic code-review skill** - focused on scraper-specific failure modes (selector rot, schema drift, anti-bot escalation, charge correctness). For general code quality, use a standard code-review skill.
