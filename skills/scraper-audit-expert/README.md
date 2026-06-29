# Scraper Audit Expert — a Claude skill

A Claude Code skill that runs a **senior-level audit of an existing scraper** — whether it's an Apify Actor or a standalone service (Express/PM2, FastAPI, cron) — and produces a structured, severity-classified report with prioritized fixes.

It surfaces what most reviews miss: untapped fields, silent data-quality leaks, consistency bugs, resilience holes, performance/cost waste, observability gaps, and architectural debt — each finding quantified with "% of records affected" or "$ at risk", never a vague hot-take.

## Why this skill exists

A scraper that's "working" is rarely as healthy as it looks. Success rate drifts when selectors rot. Untapped JSON-LD fields are invisible by definition. A currency parser that silently drops GBP symbols looks fine in code and broken in the data. Costs creep up when cache hit rate or concurrency is mistuned. And a scraper that breaks silently in production for three weeks is worse than one that crashes loudly on day one.

This skill encodes a repeatable audit methodology so any Claude Code session can review a scraper the way a careful senior engineer would — gather real artifacts first, walk eight dimensions in dependency order, quantify every critical finding from data, and deliver a report someone can actually act on.

## What it does

- **8-dimension audit framework** — Coverage, Quality, Consistency, Resilience, Performance & Cost, Observability, Architecture, and (Apify-only) Monetization, each with a concrete checklist, commands, and severity criteria.
- **Two runtimes** — distinguishes Apify Actors (Console / API artifacts) from standalone services (PM2 logs, Postgres, Redis), branching the gather step accordingly.
- **Coverage discovery** — a five-step method (field inventory, hidden-API probe, managed-API raw diff, schema.org probe, long-tail probe) to find what the scraper *could* extract but isn't.
- **Latent-bug grep catalog** — 30+ patterns (empty catch, optional chains on parsed data, `Promise.all` on independent calls, single selectors, charge-in-catch, `as any`, …) with why each is a bug and how to confirm.
- **Quality & consistency tests** — runnable SQL / jq queries for null rates, schema conformance, normalization, dedup correctness, schema drift, cross-page and cross-source agreement.
- **Severity model + report template** — `[SEV-N][Effort][category]` classification, prioritization by severity/effort, and a fillable 8–15 page report template with a worked example.

## What's inside

```
scraper-audit-expert/
├── SKILL.md                                  # Orchestration hub: 8 dimensions, severity model, 5-phase workflow
├── README.md                                 # This file
├── INSTALL.md                                # Installation instructions
└── references/
    ├── audit-framework.md                    # ⭐ Full 8-dimension checklist + commands + severity criteria
    ├── coverage-methodology.md               # Five-step systematic coverage analysis
    ├── quality-and-consistency-tests.md      # Concrete SQL/jq tests (null rates, drift, dedup, cross-source)
    ├── latent-bug-patterns.md                # 30+ grep patterns + why each is a bug + how to confirm
    └── audit-report-template.md              # Fillable report template + worked finding example
```

## Installation

See `INSTALL.md`. TL;DR for macOS/Linux:

```bash
mkdir -p ~/.claude/skills
unzip scraper-audit-expert.zip -d ~/.claude/skills/
```

## How to invoke

Once installed, the skill auto-activates on scraper-audit requests. Example prompts:

- "Audit this scraper — is it extracting all the available data?"
- "Why did my Actor's success rate drop without a code change?"
- "Review my scraper for data-quality and consistency bugs"
- "What's wrong with this scraper? Find the latent bugs."
- "My scrape cost-per-record is creeping up — where's the waste?"

If auto-activation misses, force it: "Use the `scraper-audit-expert` skill to..."

## When NOT to use it

- The scraper is **fully broken** (success rate < 30%) — that's triage, not audit. Fix the obvious failure first.
- The scraper is **being designed** (not yet implemented) — use a scraper/MCP *build* skill instead.
- The question is **purely about pricing/monetization** — use a dedicated PPE/pricing audit framework.
- The question is **purely about anti-bot** ("how do I get past Cloudflare?") — use an anti-bot strategies reference.

This skill **identifies** problems and cross-references the right doctrine for the fix — it does not duplicate fix recipes.

## Part of the mr-bridge.com toolkit

This skill is part of the [mr-bridge.com](https://mr-bridge.com) toolkit for scraping, data, and content automation. Related resources:

- [mr-bridge.com](https://mr-bridge.com) — home
- [Scrapers](https://mr-bridge.com/scrapers) — the Apify Actor portfolio
- [MCP servers](https://mr-bridge.com/mcp-servers) — Model Context Protocol servers
- [AI workflows](https://mr-bridge.com/ai-workflows) — agents and automation
- [Studies](https://mr-bridge.com/studies) — data studies and one-pagers
- [Articles](https://mr-bridge.com/articles) — write-ups and guides
- [Solutions](https://mr-bridge.com/solutions) — end-to-end solutions

## License

Personal use. Customize freely. No warranty.
