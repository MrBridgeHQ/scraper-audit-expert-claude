# Installation — Scraper Audit Expert skill

This skill is designed to be installed at the **user level** in Claude Code, so it's available across all your projects without copying it into each repo.

It is a documentation-only skill (Markdown references, no scripts), so there are no runtime dependencies to install.

## Prerequisites

- Claude Code installed and working
- A terminal with `unzip` (macOS, Linux) or PowerShell with `Expand-Archive` (Windows)

Optional, only if you want to run the audit commands the skill suggests:
- `jq` for the Apify JSON queries
- The `apify` CLI for Apify Actor audits
- `psql` / `redis-cli` for standalone (Postgres/Redis) audits

## Installation

### macOS / Linux

```bash
# 1. Create the user-level skills directory if it doesn't exist
mkdir -p ~/.claude/skills

# 2. Unzip the skill into that directory
unzip scraper-audit-expert.zip -d ~/.claude/skills/

# 3. Verify the structure
ls ~/.claude/skills/scraper-audit-expert/
# Expected: SKILL.md  README.md  INSTALL.md  references/
```

### Windows (PowerShell)

```powershell
# 1. Create the user-level skills directory if it doesn't exist
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills"

# 2. Unzip the skill into that directory
Expand-Archive -Path .\scraper-audit-expert.zip -DestinationPath "$env:USERPROFILE\.claude\skills\"

# 3. Verify the structure
Get-ChildItem "$env:USERPROFILE\.claude\skills\scraper-audit-expert\"
# Expected: SKILL.md  README.md  INSTALL.md  references/
```

## Verification

Open Claude Code and ask:

> "What skills do you have access to?"

You should see `scraper-audit-expert` in the list. If not, check that the SKILL.md file is at the path:

- macOS / Linux: `~/.claude/skills/scraper-audit-expert/SKILL.md`
- Windows: `%USERPROFILE%\.claude\skills\scraper-audit-expert\SKILL.md`

## First test

In any directory (Claude Code session), ask:

> "Audit this scraper — is it extracting all the available data, and what's the data quality?"

The skill should activate, walk the 5-phase workflow (gather artifacts → walk the 8 dimensions → quantify findings → sequence fixes → deliver the report), and produce a structured, severity-classified audit report.

## Updating the skill

To install a newer version, just replace the directory:

```bash
# macOS / Linux
rm -rf ~/.claude/skills/scraper-audit-expert
unzip scraper-audit-expert.zip -d ~/.claude/skills/
```

## Uninstalling

```bash
# macOS / Linux
rm -rf ~/.claude/skills/scraper-audit-expert

# Windows (PowerShell)
Remove-Item -Recurse -Force "$env:USERPROFILE\.claude\skills\scraper-audit-expert"
```

## Troubleshooting

**Skill doesn't activate when expected.**
The skill's auto-activation depends on the description in `SKILL.md`'s frontmatter. Triggers include: "audit this scraper", "review my scraper", "what's wrong with X", "is this scraper extracting all available data", "find bugs in scraper", "why does scraper Y fail". If your phrasing doesn't match, force activation: "Use the `scraper-audit-expert` skill to..."

**The audit commands reference tools I don't have.**
The skill's commands (`apify`, `jq`, `psql`, `redis-cli`) are illustrative — install only the ones relevant to your runtime. The framework itself is tool-agnostic; the skill notes workarounds when a tool is missing.
