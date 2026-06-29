# Audit Report Template

Fillable template for the audit deliverable.

**Length discipline:** 8–15 pages total. Cut to top 20 findings; appendix the rest.

**Storage:** the audit report is **internal documentation** (it mentions findings, impact estimates, and business context). It MUST NOT live inside an Actor's git repository that gets `apify push`-ed, because `apify push` uploads the whole repo and the public API exposes `versions[0].sourceFiles[]`. Store it outside the Actor's source tree — e.g. a separate private repo or a private docs folder.

---

## Template

```markdown
# Scraper Audit — <Project name>

**Date:** <YYYY-MM-DD>
**Auditor:** <Claude with skill `scraper-audit-expert`>
**Scope:** <list of Actors / scraper services audited>
**Runtime:** <Apify Actor | Standalone (PM2/etc.) | Both>
**Sources of evidence:**
- <e.g. `apify actors info` API, last 30 days of runs>
- <e.g. Postgres `scraped_records` last 7 days (N=23,400 records sampled)>
- <e.g. Source code at commit <sha>>
- <e.g. Production logs `pm2 logs <process> --lines 10000`>

---

## Executive summary

<5–7 bullets. Each bullet is a single concrete finding or a single concrete metric.>

- Overall health: <🔴 Critical / 🟡 Important / 🟢 Healthy>. Justification: <one sentence>.
- Top finding: <single-sentence summary of the worst issue, with impact>.
- Quick-wins available: <N> findings at SEV 3-4 effort S. Estimated <X> days of work, <Y%> success-rate gain.
- Strategic work: <N> findings requiring architectural change. Estimated <X> weeks.
- Cross-skill referrals: <e.g. "3 findings delegate to a pending PPE/pricing audit">

---

## Methodology & scope

**What was looked at:**
- Source tree at <commit sha> / branch `<name>`
- <N> sample records from <runtime>, drawn from <date range>
- <X> sample target pages reviewed for coverage gaps (DevTools-based)
- Last <Y> runs analyzed via Apify Console / PM2 logs

**What was NOT looked at:**
- <e.g. "Frontend code (out of audit scope)">
- <e.g. "An upstream API client (separate audit)">
- <e.g. "Tests directory — limited to verifying fixture coverage, not test logic">

**Sample size justification:** <state confidence interval where relevant>. Example: "N=200 randomly-sampled records, accuracy claims have ±7% confidence at 95%."

**Time window:** <e.g. "Production data from 2026-04-27 to 2026-05-27 (30 days)">.

---

## Snapshot — scrapers in scope

**Multi-target audit note:** if auditing >1 scraper in the same engagement, produce **one unified report** with both Snapshot subsections below populated. Findings are still numbered globally (C1, C2, ...) but each finding's header includes the affected scraper(s): `[SEV-N][Effort][category] (Scraper A) Title` or `(both)` for cross-cutting findings. Methodology + scorecard sections collapse to a single instance; only Snapshot tables and finding blocks branch by scraper.

### Runtime: Apify Actor

| Actor | Model | Recent runs (30d) | Success rate | Mean items/run | Memory tier | Last build |
|---|---|---|---|---|---|---|
| `<user>/<slug-1>` | PPE $X | <N> | <%> | <M> | <MB> | <date> |
| `<user>/<slug-2>` | PPE $X | <N> | <%> | <M> | <MB> | <date> |
| ... | ... | ... | ... | ... | ... | ... |

### Runtime: Standalone

| Service | Process | Target sources | Records / day | Uptime | Last restart |
|---|---|---|---|---|---|
| `<name>` | PM2 `<process>` | <e.g. source A, source B, source C> | <N> | <%> | <date> |

---

## Dimension scorecard

| Dimension | Score | One-line justification |
|---|---|---|
| 1. Coverage | <N>/5 | <e.g. "JSON-LD probe found 4 untapped fields; coverage on long-tail edge cases inconsistent."> |
| 2. Quality | <N>/5 | <e.g. "Null rate <2% on critical fields. Currency normalization fails on GBP at 12%."> |
| 3. Consistency | <N>/5 | <e.g. "Cross-run stability OK. Schema drift detected on `availability` enum starting week of 2026-04-15."> |
| 4. Resilience | <N>/5 | <e.g. "Single-selector fields on price extraction. `Actor.fail` called for business errors."> |
| 5. Performance & cost | <N>/5 | <e.g. "Cost-per-record $0.0004, well under PPE $0.005 — healthy margin. Cache hit rate 32%, low."> |
| 6. Observability | <N>/5 | <e.g. "Structured logs ✓. No alerting on success-rate drops."> |
| 7. Architecture | <N>/5 | <e.g. "Clean separation, fixtures present, 18 test files. 4 `as any` casts to address."> |
| 8. Monetization | Delegated | See <path to PPE/pricing audit dated YYYY-MM-DD> |

**Scoring scale (general):**
- 5/5 — best-in-portfolio, no actionable findings
- 4/5 — healthy with minor optimizations
- 3/5 — functional but real findings exist
- 2/5 — material issues affecting users or revenue
- 1/5 — critical failures or fundamental gaps

**Per-dimension scoring exemplars** (use to calibrate):

| Dim | 5/5 | 3/5 | 1/5 |
|---|---|---|---|
| 1. Coverage | 0 JSON-LD gaps, all 14 long-tail edge cases handled, hidden API consumed if exists | 1–3 gaps on routine fields, long-tail edges inconsistent | Scraper bypasses documented hidden API; >5 fields missing from JSON-LD |
| 2. Quality | <1% null on non-nullable fields, schema validates >99%, all normalization checks pass | 2–10% null on some fields, minor normalization drift | >20% null on non-nullable, mixed currency symbols persisted as strings, junk records present |
| 3. Consistency | 0% name changes on stable URLs, 0 duplicates, no schema drift detected | Some cross-page disagreement, occasional new enum values | Step changes in null rate ignored, dedup broken, schema drift undetected for >30 days |
| 4. Resilience | All critical fields multi-selector, graceful exit pattern, p-limit semaphores, captcha guards | Some single-selector fields, generic retries | `Actor.fail()` for business errors, no cascade containment, captcha returned as data |
| 5. Perf & Cost | Cost/record well under PPE price, cache hit ≥80%, memory plateau healthy | Cache hit 40–60%, memory tier slightly oversized | Cost/record > event price, cache hit <20%, sequential where parallel safe |
| 6. Observability | Structured logs, full metric set, alerting on success rate + cost, runbooks | Logs only, no alerting, metric gaps | Free-form logs, no error records, blind on cost |
| 7. Architecture | Pure extractors, fixtures present, 0 `as any`, secrets clean | Some `as any`, partial fixture coverage, occasional doc drift | Secrets in repo, no tests, monolithic single-file scraper |
| 8. Monetization | (Delegated — see the PPE/pricing audit) | (Delegated) | (Delegated) |

---

## Findings 🔴 Critical

### C1 — <Title>

**[SEV-N][Effort][category]**

**Diagnostic:**
<What's wrong. 2-4 sentences. Include file:line, query output, or sample bad record.>

```
<code excerpt OR sample bad record OR query output>
```

**Impact:**
<Quantified. % of records, $ at risk, users affected, success-rate impact. Show the math.>
- Affected records: <N> over last 30 days (`<query>`)
- Estimated revenue impact: <$Y/month> based on <calculation>
- User-facing symptom: <what they see>

**Fix direction:**
<1–3 sentences. Cross-reference the fix doctrine. Do not write the full code.>

> Cross-ref: <doctrine reference> § "<section>"

**Confidence:** <High / Medium / Low — based on data quality used to compute impact>

---

### C2 — <Title>

<same structure>

---

### C3 — <Title>

<same structure>

---

## Findings 🟡 Important

### I1 — <Title>

**[SEV-3][Effort][category]**

**Diagnostic:** <2-3 sentences>

**Impact:** <quantified or labeled "Inferred (no direct measurement available)">

**Fix:** <one sentence, cross-ref>

---

### I2 — <Title>

<same structure, condensed>

---

## Findings 🟢 Nice-to-have

Bullet list, no separate sections:

- N1 — [SEV-1][S][arch] <Title> — <one line>. Fix: <one line>.
- N2 — [SEV-2][S][obs] <Title> — <one line>. Fix: <one line>.
- ...

---

## Quick wins — "Fix this week"

Sorted by severity / effort (SEV-3+, effort S only).

| Finding | Effort | Impact | Cross-ref |
|---|---|---|---|
| C1 — <Title> | S | <quick impact statement> | <doctrine ref> |
| I1 — <Title> | S | <quick impact statement> | <doctrine ref> |
| ... | | | |

---

## Strategic recommendations — "Fix this quarter"

Sorted by severity (SEV-4+, effort M/L).

| Finding | Effort | Impact | Cross-ref |
|---|---|---|---|
| C3 — <Title> | M | <quick impact statement> | <doctrine ref> |
| ... | | | |

---

## Open questions / Needs more data

Findings where impact could not be quantified from available data, OR where the audit lacks evidence to conclude. **This section is non-negotiable — never skip.**

- Q1 — <Question>. To resolve: <what data / access would unblock>. Example: "Cache hit rate not measurable — no `redis-cli INFO stats` access during audit window. Resolution: instrument or grant access."
- Q2 — <Question>. To resolve: <...>.
- ...

---

## Sequencing the fixes

**Apify 14-day constraint:** <if applicable — list which fixes require a 14-day delay; sequence one delayed change per Actor per month>.

**Dependency constraint:** <if applicable — list which fixes must precede others. Example: "Add observability for cache hit rate (I4) BEFORE optimizing cache TTL (S2)">.

Proposed sequence:

| Week | Fixes |
|---|---|
| Week 1 (immediate) | C1, I1, I3 |
| Week 2 (immediate) | C2, I2, N1, N2, N3 |
| Week 3 (delayed start) | C3 (14-day delay) |
| Week 4–6 | S1, S2 (strategic) |

---

## Appendix

### A.1 — Raw queries used

```sql
-- Q1 (null rate per field)
SELECT
  100.0 * COUNT(*) FILTER (WHERE price IS NULL) / COUNT(*) AS pct_null_price
FROM scraped_records WHERE created_at > NOW() - INTERVAL '7 days';

-- C3 (schema drift)
SELECT DATE_TRUNC('week', created_at) AS week, ...
```

### A.2 — Sample bad records

```json
[
  { "id": "abc123", "name": "...", "price": null, "url": "...", "issue": "..." },
  ...
]
```

### A.3 — Full null-rate table (all fields)

| Field | pct_null |
|---|---|
| name | 0.0% |
| price | 12.3% |
| ... | ... |

### A.4 — Coverage probe — JSON-LD fields found vs. extracted

| JSON-LD field | Present on source | Extracted to record |
|---|---|---|
| `Product.name` | 20/20 | ✓ |
| `Product.aggregateRating` | 17/20 | ✗ (gap C5) |
| ... | ... | ... |

---

**End of audit report.**
```

---

## Notes on filling the template

### Tone

- **Direct, technical, no hedging on facts.** "Field X is null on 12% of records."
- **Explicit hedging on inferences.** "This *likely* costs ~$Y/month — Inferred."
- **No blame language.** Findings are about the system, not engineers.

### Length

- 1 finding ≈ ½ page including diagnostic + impact + fix.
- Critical section: 3–5 findings = 2–3 pages.
- Important section: 5–10 findings = 2–3 pages condensed.
- Nice-to-have: 5–15 bullets = ½ page.
- **Total target: 8–15 pages.** Cut findings if you exceed.

### What to include in Diagnostic

- File path and line number, OR
- Query output (3–10 lines), OR
- Sample bad record (1–3 JSON examples), OR
- Code excerpt (≤10 lines)

Anything more verbose → Appendix.

### What to include in Impact

- **Quantified** (preferred): "% of records," "$ at risk," "X users complained."
- **Inferred** (acceptable with label): "Inferred ~$Y/month based on <calculation>."
- **Not measurable** (move to Open Questions): "Cannot quantify from available data."

### What to include in Fix direction

- 1–3 sentences max.
- **Cross-reference** the fix-doctrine reference (don't write the full code).
- Exception: trivial one-liner fixes (typos, type fixes) can include the patch inline.

### What to leave out

- Full code (cross-ref the build doctrine instead)
- Long architectural essays (write a separate design doc if needed)
- Findings without impact assessment (move to Open Questions instead)
- Anything not actionable (an audit is not a brainstorm)

### Output format

Markdown. PDF export optional for sharing with non-technical stakeholders (`pandoc audit.md -o audit.pdf` works fine).

For storage: keep it **outside** the Actor's own git repo if that repo is pushed to Apify (otherwise it leaks via `sourceFiles[]`). A separate private repo or private docs folder is fine.

---

## Worked example — one full finding block

This is what a complete 🔴 Critical finding looks like end-to-end. Use it to calibrate the level of detail expected.

```markdown
### C2 — Currency parsing fails for GBP listings — silent quality leak

**[SEV-4][S][quality]** (Scraper B)

**Diagnostic:**
The detail-page extractor at `backend/src/services/scraper/html-parser.ts:147` reads price text with `parsePrice(priceText)` but the regex `/[\d,.]+/` strips the currency symbol without recording it. When the original element contains `£42.50`, the parser stores `price: 42.50` and falls through to currency = `null`. EUR/USD listings are not affected because Schema.org `[itemprop="priceCurrency"]` is correctly parsed on those rows.

Sample bad record (from Postgres):
```json
{ "id": "rec-42f1...", "name": "Trail Running Jacket", "price": 1850.00, "currency": null, "country_code": "GB", "url": "https://..." }
```

**Impact:**
- Query: `SELECT 100.0 * COUNT(*) FILTER (WHERE currency IS NULL AND country_code = 'GB') / COUNT(*) FILTER (WHERE country_code = 'GB') AS pct FROM scraped_records WHERE created_at > NOW() - INTERVAL '30 days';` → **12.3%** of UK records (1,847 of 15,012 over 30 days)
- Downstream: frontend displays the product without currency, defaulting to EUR rendering — users see prices that look 1.2× too high (GBP→EUR rate)
- Confirmed via 2 user-reported tickets in the project's issue tracker (2026-04)

**Fix direction:**
In the detail-page parser, detect currency from the price text before stripping. Add a `£|€|\$` capture group ahead of the numeric regex, map symbol → ISO 4217 code (an existing country-code → currency map can be reused as a fallback). Cross-ref your selector-strategy doctrine § "Normalization" for the canonical pattern. Estimated effort: 1–2 hours including a unit test on a GBP fixture.

**Confidence:** High — query result is exact, sample matched 5/5 manual checks against live source pages.
```

---

## Quantification discipline — before/after

The most common audit mistake (per `SKILL.md` § "Common auditor mistakes" row 1) is reporting findings without quantified impact. Compare:

**❌ Bad (noise):**
> "The currency parsing might affect some users. This could lead to display issues on the frontend. Recommend reviewing the parser."

**✅ Good (finding):**
> "12.3% of UK records have `currency: null` (query above, last 30 days, N=15,012 GB-tagged records). Downstream symptom: frontend renders prices as EUR-by-default, inflating displayed cost ~1.2×. Two user tickets confirm impact. Confidence: High."

The difference is the difference between a hot take and a billable finding. **Every 🔴 finding must read like the second version.** If you can't get there, the finding belongs in "Open questions / Needs more data" with the specific query or sample that would unlock it.

---

## Cross-skill handoff pattern (Dim 8 / monetization findings)

When the audit surfaces a PPE/monetization issue, the operational reality is:

1. **In this skill's report**: write a **stub finding block** with `[SEV-N][Effort][monetization]` header, the Diagnostic from this audit, and a **one-line Impact** + Fix that says `Delegated to a PPE/pricing audit dated <YYYY-MM-DD> for the full Diagnostic/Impact/Fix block — see <path>`. Do not re-derive the PPE-correct charging pattern here.
2. **Run a dedicated PPE/pricing audit separately** if not already done in the past 30 days. Its multi-dimension audit covers the deep PPE doctrine.
3. **In the Sequencing section**, reference both reports — the 14-day rule and one-change-per-month constraint applies to whichever recommendations land first.

Some findings DOUBLE-TAG: `[SEV-5][resilience+monetization]` is correct when, e.g., `Actor.fail()` for business errors AND that path also fires `Actor.charge()` (refund magnet). Treat as one finding here with both category tags; cross-ref both doctrines.
