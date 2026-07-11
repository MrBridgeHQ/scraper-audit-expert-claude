# scraper-audit-expert - Claude Code Agent Skill

Senior-level audit of an existing scraper to find what is wrong, what is missing, and what could be optimized. An Agent Skill **for Claude Code**. It produces a structured report (executive summary, scorecard, prioritized findings, and fixes), not a stream of "consider X". It works for Apify Actors and standalone scrapers alike (Express/PM2, FastAPI, cron).

Two convictions drive it: every dimension matters equally until proven otherwise (do not skip coverage because "the data looks fine"), and a finding without quantified impact is noise (every critical finding states the percent of records affected, the dollars at risk, or the users blocked).

## The 8 dimensions

| # | Dimension | What it answers |
|---|---|---|
| 1 | Coverage | What the scraper could extract vs what it actually does: untapped fields, missing pagination, missing entity types. |
| 2 | Quality | Null rates, accuracy vs ground truth, schema conformance, normalization (currency, dates, encoding). |
| 3 | Consistency | Cross-run stability, cross-page agreement, dedup correctness, schema drift over time. |
| 4 | Resilience | Selector fallbacks, graceful exit, anti-bot escalation, timeout handling, cascade containment. |
| 5 | Performance and cost | Throughput, latency p50/p95/p99, cost per successful record, cache hit rate, concurrency tuning. |
| 6 | Observability | Structured logs, emitted metrics, alerting on degradation, single-record debuggability. |
| 7 | Architecture | Code organization, testability, type safety, fixture coverage, secrets hygiene. |
| 8 | Monetization (Apify) | Delegated to a dedicated pricing audit, not re-audited here. |

## When to use

- A scraper is "working" but you suspect data-quality issues, or its success rate dropped with no code change (selector rot, site change, anti-bot escalation).
- Costs are creeping up (managed-API or Apify compute) and you want the cache and concurrency examined.
- A new maintainer needs to know what the scraper actually does versus what it claims.
- A pre-launch or post-incident review of the surface area.

## When NOT to use

- The scraper is fully broken (success rate under 30 percent): that is triage, not audit. Fix the obvious failure first.
- The scraper is being designed, not yet built: use a build skill.
- The question is purely pricing or purely anti-bot: use the dedicated pricing-audit or anti-bot references.

## Output

A severity-classified report: an executive summary, a per-dimension scorecard, findings ranked by impact (critical, high, medium), and prioritized, concrete fixes. It distinguishes the two runtimes (Apify Actor vs standalone) at the gather-artifacts step, with the exact commands to run for each.

## Installation

### Claude Code, project level

```bash
cp -r skills/scraper-audit-expert /path/to/your-project/.claude/skills/
```

### Claude Code, global level

```bash
cp -r skills/scraper-audit-expert ~/.claude/skills/
```

## Use it

- "Audit this scraper and tell me what is wrong, ranked by impact."
- "Is this scraper extracting all the available data, or leaving fields on the table?"
- "My success rate dropped from 96 to 70 percent with no code change, what happened?"
- "What is my cost per successful record, and how do I bring it down?"
- "Review this Actor for data quality, resilience, and observability before launch."

## What is inside

The skill lives in [`skills/scraper-audit-expert/`](skills/scraper-audit-expert/): a `SKILL.md` plus a `references/` library (the full per-dimension audit framework with concrete commands, the severity classification, and the report template).

## License

See `LICENSE`.

---

Part of the **[mr-bridge.com](https://mr-bridge.com)** toolkit for scraping, data, and content automation:
[Scrapers](https://mr-bridge.com/scrapers) · [MCP servers](https://mr-bridge.com/mcp-servers) · [AI workflows](https://mr-bridge.com/ai-workflows) · [Studies](https://mr-bridge.com/studies) · [Articles](https://mr-bridge.com/articles) · [Solutions](https://mr-bridge.com/solutions)

---

*Part of the [MrBridge Agent Skills catalog](https://github.com/MrBridgeHQ/skills). Browse them all at [mr-bridge.com/skills](https://mr-bridge.com/skills).*
