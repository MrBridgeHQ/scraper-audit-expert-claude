# Latent Bug Patterns - Grep Catalog

The third systematic methodology (after coverage + quality tests). Latent bugs don't show up in logs and don't fail tests - they silently degrade data quality.

This file gives **30+ grep patterns** + why each is a bug + how to confirm. Each pattern produces **candidates** - confirm by reading context before filing a finding.

Greps assume TypeScript scrapers (Node + TS dominant). Python equivalents noted where syntax differs significantly.

---

## Tier 1 - Silent swallow patterns

These hide bugs by catching errors and continuing as if nothing happened.

### L1 - Empty catch blocks

```bash
# Node/TS
grep -rEn 'catch\s*\([^)]*\)\s*\{\s*\}' src/
grep -rEn 'catch\s*\{\s*\}' src/

# Python
grep -rEn 'except[^:]*:\s*$\s*pass' --include='*.py' src/
```

**Why:** The error happens, gets caught, and nothing happens. The calling code believes the operation succeeded. Silent data loss or junk records.

**Confirm:** read the catch block. If the code intentionally swallows (e.g. logging-only retry inner loop), it should be `catch (e) { log.warn(e) }` not `catch {}`.

**Finding:** `[SEV-4][S][resilience] Empty catch at <file>:<line> - silently swallows errors`

### L2 - Catch with only `console.log` / `log.info`

```bash
grep -rEn -A1 'catch\s*\(' src/ | grep -B1 'log\.(info|warn|debug)\|console\.' | head -20
```

**Why:** Logs are easy to miss. Errors should either propagate, push to dataset as error records, or both.

**Confirm:** does the calling code know the operation failed? If not, error swallowed.

**Finding:** `[SEV-3][S][resilience] Catch at <file>:<line> only logs - caller not informed of failure`

### L3 - Optional chain on parsed data

```bash
grep -rEn '(data|response|result|item|json)\?\.\w+\?\.' src/
grep -rEn '\w+\?\.\w+\?\.\w+' src/   # 3-level optional chain
```

**Why:** `data?.price?.value` makes a parsing failure look like a missing field. The record is persisted with `price = null` and looks like an out-of-stock product when it's actually a parser bug.

**Confirm:** is the path guaranteed by the source's schema? If yes, the optional chain is hiding bugs. If no, the optional chain is correct but should be paired with a fallback selector.

**Finding:** `[SEV-3][S][quality] Optional chain at <file>:<line> hides parser failures - convert to explicit null check + alternate path`

### L4 - `|| null` / `|| 0` / `|| ''` defaults on extraction

```bash
grep -rEn '\|\|\s*(null|0|'\'\'\''|""|\[\])' src/extractors/ src/services/scraper/
grep -rEn '\?\?\s*(null|0|'\'\'\''|""|\[\])' src/extractors/ src/services/scraper/
```

**Why:** `extractPrice($) || null` looks defensive but turns "selector returned undefined" into "field doesn't exist". The downstream system can't tell the difference. Bug becomes data.

**Confirm:** if `extractPrice` returns valid value (including 0), is `|| 0` masking it? Numeric zero ≠ missing.

**Finding:** `[SEV-3][S][quality] Default-coalesce at <file>:<line> hides extraction failures`

### L5 - Try/catch around the entire request handler

```bash
grep -rEn -B1 -A10 'requestHandler.*async' src/routes.ts | grep -A8 'try'
```

**Why:** Catching everything at the top of a handler swallows specific error categories that should be distinguished (anti-bot vs. parser vs. network).

**Confirm:** does the catch differentiate by error class? If just `catch (e) { log.error(e) }`, see L2.

---

## Tier 2 - Anti-pattern selectors

### L6 - Hardcoded single-selector (no fallback)

```bash
# Looks for $('selector') with no comma (= no fallback)
grep -rEn '\$\(['\''"][^,'\''"]+['\''"]\)' src/extractors/
```

**Why:** Single-selector fields break the first time the target tweaks class names.

**Confirm:** is the selector anchored on stable markup (e.g. `[itemprop="name"]`)? If on a visual class (`.title-text`), it's brittle.

**Finding:** `[SEV-3][S][resilience] Single-selector `<sel>` at <file>:<line> - add fallback to schema.org or alternate class`

### L7 - Visual class selectors instead of schema.org

```bash
# Visual classes
grep -rEn '\$\(['\''"]\.(price|name|title|description|rating)' src/
# vs. structured markup (good)
grep -rEn 'itemprop=' src/   # should appear frequently
```

**Why:** Visual classes change with redesigns; schema.org `itemprop` is stable across years.

**Finding:** `[SEV-3][S][resilience] Selector uses visual class `<sel>` at <file>:<line> - prefer schema.org`

### L8 - String concatenation for URLs (instead of `new URL`)

```bash
grep -rEn '['\''"]https?://['\''"] \+|\+ ['\''"]/'\''"]' src/
grep -rEn '\$\{baseUrl\}\$\{' src/
```

**Why:** `baseUrl + href` breaks on absolute hrefs (`https://...`) and on protocol-relative (`//cdn...`). `new URL(href, baseUrl).href` handles all cases.

**Finding:** `[SEV-3][S][quality] URL concatenation at <file>:<line> - use \`new URL(href, baseUrl).href\``

### L9 - `.text()` without `.trim()` / normalization

```bash
grep -rEn '\.text\(\)$' src/extractors/   # only `.text()` at end of line
grep -rEn '\.text\(\)\.replace\(' src/   # ad-hoc cleaning
```

**Why:** `.text()` returns raw text including whitespace, newlines, HTML entities. Persisted as-is = inconsistent data.

**Finding:** `[SEV-2][S][quality] `.text()` without trim/normalize at <file>:<line>`

---

## Tier 3 - Concurrency and async pitfalls

### L10 - `Promise.all` on independent extraction calls

```bash
grep -rEn 'Promise\.all\(\[.*await' src/
grep -rEn -B1 'Promise\.all\(\[' src/ | grep -A1 'runAgent\|fetch\|scrape\|extract'
```

**Why:** `Promise.all` rejects on the first failure. In a 6-agent pipeline, one failure kills the whole run. `Promise.allSettled` lets you handle partial success.

**Confirm:** are the items independent? If yes → should be `allSettled`. If they have ordered dependencies → `Promise.all` is correct.

**Finding:** `[SEV-4][S][resilience] `Promise.all` on independent calls at <file>:<line> - use `Promise.allSettled`. Cross-ref your error-handling doctrine § "Multi-agent and parallel patterns"`

### L11 - No semaphore on outbound requests

```bash
grep -rEn 'pLimit\|p-limit' src/
# If absent or only on one provider, third-party APIs may be unrate-limited
```

**Why:** Unbounded concurrency triggers 429s or burns paid quota. A typical pattern uses `p-limit` with a small slot count per provider (e.g. 3 for a heavy browser API, 5 for an HTML-scraping API) - replicate it across every outbound dependency.

**Finding:** `[SEV-3][M][resilience] Missing concurrency cap on <outbound API> at <file>:<line>`

### L12 - Synchronous `fs.readFileSync` / `writeFileSync` in hot path

```bash
grep -rEn 'readFileSync\|writeFileSync' src/
# Check if any are inside request handlers, routes, or scrapers
```

**Why:** Blocks the event loop. Throughput tanks on busy paths.

**Confirm:** is the sync I/O at startup (acceptable) or per-request (bad)?

**Finding:** `[SEV-3][S][perf] Synchronous I/O in hot path at <file>:<line>`

### L13 - `await` inside `for` loop (sequential where parallel works)

```bash
grep -rEn -B1 'for\s*\(.*of' src/ | grep -A1 'await'
```

**Why:** Often unintentional sequentialization. Should be `await Promise.all(items.map(async ...))` when iterations are independent.

**Confirm:** does each iteration depend on the previous? If no → parallelize.

**Finding:** `[SEV-2][S][perf] Sequential await loop at <file>:<line> - consider `Promise.all`/`Promise.allSettled``

---

## Tier 4 - Cache misuse

### L14 - Cache write without TTL or invalidation hook

```bash
grep -rEn 'setValue\|set\(' src/services/scraper/ | grep -v 'ttl\|TTL\|expiry'
grep -rEn 'redis\..*\.set\(' src/ | grep -v 'EX\|expire'
```

**Why:** Stale data forever. Apify KV has no native TTL; standalone Redis needs explicit `EX`/`expire`.

**Finding:** `[SEV-3][S][quality] Cache write without TTL at <file>:<line>`

### L15 - Cache read without staleness check

```bash
grep -rEn 'getValue\|get\(.*cache' src/ | head -10
# Pair with grep for timestamp comparison
```

**Why:** If TTL is code-managed via stored timestamps, the read must check `Date.now() - entry.ts < TTL_MS`. Missing check = serving forever-stale data.

**Finding:** `[SEV-3][S][quality] Cache read at <file>:<line> doesn't check staleness`

### L16 - Cache key not deterministic / depends on object iteration order

```bash
grep -rEn 'JSON\.stringify\(.*\)' src/services/scraper/cache* src/services/scraper/*cache*
```

**Why:** `JSON.stringify({a:1, b:2})` and `JSON.stringify({b:2, a:1})` produce different strings. Cache misses on logically-identical inputs.

**Confirm:** is the input always built in the same order? If user-supplied object → use a sorted-key stringify or hash of sorted entries.

**Finding:** `[SEV-2][S][quality] Cache key relies on JSON.stringify order at <file>:<line>`

---

## Tier 5 - PPE / billing pitfalls (Apify)

These are also covered in a dedicated PPE anti-patterns doctrine. The audit cross-references; here we surface candidates.

### L17 - `Actor.charge` in catch block

```bash
grep -rEn -A1 'catch\s*\(' src/ | grep -B1 'Actor\.charge'
```

**Why:** Charges on error path = refund requests. See a PPE doctrine § "charge after success only".

**Finding:** `[SEV-5][S][monetization] Actor.charge inside catch at <file>:<line> - cross-ref a PPE doctrine`

### L18 - `Actor.charge` without `eventChargeLimitReached` check

```bash
grep -rEn 'Actor\.charge' src/ | grep -v 'eventChargeLimitReached\|chargeResult\|charge\.event'
```

**Why:** Free-plan users get infinite responses while the creator earns $0. See a PPE doctrine § "charge limit".

**Finding:** `[SEV-4][S][monetization] Actor.charge without ChargeResult check at <file>:<line>`

### L19 - `Actor.fail()` for business-logic errors

```bash
grep -rEn 'Actor\.fail\(' src/
# Each occurrence should be reviewed - only acceptable in true infra-error paths
```

**Why:** Tanks success rate. See your error-handling / PPE doctrine.

**Finding:** `[SEV-5][S][resilience] Actor.fail() for business error at <file>:<line>`

### L20 - Double-charging via duplicate event declarations

Check `.actor/pay_per_event.json` for both `apify-default-dataset-item` AND a custom event. If both: every `pushData` charges both.

```bash
jq 'keys' .actor/pay_per_event.json
```

**Finding:** `[SEV-5][S][monetization] Double-charge: both `apify-default-dataset-item` and custom event declared - cross-ref a PPE anti-patterns doctrine`

---

## Tier 6 - Type safety waivers

### L21 - `as any` casts

```bash
grep -rEn 'as any\|<any>' src/ | grep -v 'tests/\|\.d\.ts'
```

**Why:** Defeats TypeScript's purpose. Often hides parsing failures.

**Confirm:** is there a justified reason (third-party SDK with bad types)? If no, fix.

**Finding:** `[SEV-2][S][arch] `as any` at <file>:<line> - narrow the type`

### L22 - `@ts-ignore` / `@ts-expect-error`

```bash
grep -rEn '@ts-(ignore|expect-error|nocheck)' src/
```

**Why:** Each is a place where TypeScript caught a real bug and was told to ignore it.

**Finding:** `[SEV-2][S][arch] TypeScript suppression at <file>:<line> - investigate or document`

### L23 - Missing schema validation at boundaries

```bash
# Look for direct return without Zod/Schema validation
grep -rEn 'return.*json\(\)\|return.*await.*response' src/services/scraper/
# Should be wrapped: ProductSchema.parse(await response.json())
```

**Why:** Bad data from upstream propagates everywhere.

**Finding:** `[SEV-3][S][quality] Missing schema validation at boundary <file>:<line>`

---

## Tier 7 - Configuration drift

### L24 - Hardcoded URLs in source

```bash
grep -rEn '['\''"]https?://[^'\''"]+['\''"]' src/ | grep -v 'tests/\|fixtures/'
```

**Why:** URLs in source mean config changes require code redeploys. Should be in env / `actor.json` / config files.

**Finding:** `[SEV-2][M][arch] Hardcoded URL at <file>:<line>`

### L25 - Hardcoded selectors (instead of constants file)

```bash
# Look for inline selectors deep in code (vs. centralized constants.ts)
grep -rEn '\$\(['\''"]\..+['\''"]\)' src/ | grep -v 'constants\.ts\|selectors\.ts'
```

**Why:** When the target redesigns, scattered selectors mean N edits. Centralized = 1 edit.

**Finding:** `[SEV-2][M][arch] Inline selector at <file>:<line> - centralize in src/constants.ts`

### L26 - Secrets in actor.json / source

```bash
# Look for token-like strings
grep -rEn '(apify_api_|sk-|api[_-]?key|token).*['\''"][a-zA-Z0-9_-]{20,}['\''"]' src/ .actor/
```

**Why:** Apify `apify push` uploads everything - secrets land in `sourceFiles[]`.

**Finding:** `[SEV-5][S][arch] Possible secret at <file>:<line> - move to Apify Console env vars`

---

## Tier 8 - Captcha / interstitial misclassification

### L27 - No guard before parsing scrape response

```bash
# Look for parseHtmlPage / extract directly on response without precheck
grep -rEn 'parseHtmlPage\|extractData' src/ | grep -v 'hasUseful\|isValid\|isBlocked'
```

**Why:** If the target returns a Cloudflare interstitial (200 status), parsing it produces junk records. A robust scraper has a `hasUsefulData()` guard - replicate the pattern.

**Finding:** `[SEV-4][S][resilience] Parser called without validity guard at <file>:<line> - add `hasUseful*()` precheck`

### L28 - Cookie banner / overlay treated as content

This is rarer but real. If the scraper extracts `name` from the first matching selector and the first match is "We use cookies" or "Sign up for newsletter", all records carry that text.

**Confirm:** sample 100 records, look for repeated text matches across unrelated URLs.

**Finding:** `[SEV-4][S][quality] Universal cookie banner text leaks into `<field>` - anchor selector on stable element`

---

## Tier 9 - Observability gaps

### L29 - `console.log` instead of structured logger

```bash
grep -rEn 'console\.(log|info|warn|error)' src/ | grep -v 'tests/'
```

**Why:** Unstructured logs are unsearchable. Use Apify's `log` namespace or pino/winston in standalone.

**Finding:** `[SEV-2][S][obs] console.log at <file>:<line> - use structured logger`

### L30 - Missing trace/request ID propagation

```bash
grep -rEn 'requestId\|traceId\|correlationId' src/   # should appear if request tracing is implemented
```

**Why:** Multi-step debugging is impossible without trace IDs across services.

**Finding:** `[SEV-3][M][obs] No request/trace ID propagation in <flow> - cross-service debugging blocked`

### L31 - No status message terminal flag (Apify)

```bash
grep -rEn 'setStatusMessage' src/ | grep -v 'isStatusMessageTerminal'
```

**Why:** Status messages without `isStatusMessageTerminal: true` get overwritten by the platform's default at run end.

**Finding:** `[SEV-2][S][obs] setStatusMessage at <file>:<line> missing isStatusMessageTerminal flag`

---

## How to use this catalog

1. **Run greps in order** - Tier 1 (silent swallows) first, they hide the worst bugs.
2. **Confirm by reading context** - each grep produces candidates, not findings.
3. **Cross-reference fix doctrine** - every finding links to the build skill or doctrine reference for the recipe.
4. **Cap at top 20** - if you find more than 20 latent bugs, write a follow-up audit specifically for code quality.

## What this catalog does NOT cover

- **Race conditions** require deeper analysis than grep - see Phase 2 § Resilience checklist
- **Memory leaks** require profiling - out of scope for static analysis
- **Logic bugs specific to the target site** require domain knowledge - outside this skill's scope
- **Build-system bugs** (Dockerfile, package.json) - out of scope; covered in build-skill checklists
