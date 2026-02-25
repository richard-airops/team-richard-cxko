# Fact Checker

> An article fact checker that takes a drafted article and systematically fact-checks every claim — cross-referencing SERP results, the brand's knowledge base, and the original source material. Outputs a report flagging anything unverifiable, outdated, or hallucinated with suggested corrections. Then optionally applies those corrections to produce a clean updated article.

## Overview

This project provides a two-phase fact-checking and correction pipeline. Phase 1 (`/fact-check`) analyzes an article and produces a verification report. Phase 2 (`/apply-fact-check`) reads that report, researches current data, and applies corrections — outputting a changelog and updated article.

## Quick Start

| Command | What it does |
|---------|--------------|
| `/fact-check [url]` | Scrape article, extract claims, cross-reference KB + web, generate verification report |
| `/apply-fact-check` | Read saved report, research corrections, apply updates, output changelog + corrected article |
| `/save [url]` | Save session work to `outputs/sessions/` |

## How It Works

### Phase 1: Fact-Check

```
User: "/fact-check https://www.servicetitan.com/licensing/hvac/iowa"
  ↓
Skill: fact-check/SKILL.md executes
  ↓
Step 1: Scrape URL → clean article text
  ↓
Step 2: Extract all claims, data points, stats
  ↓
Step 3: Cross-reference AirOps Knowledge Base (MCP)
  ↓
Step 4: Deep web research verification
  ↓
Step 5: Flag outdated / incorrect / hallucinated
  ↓
Step 6: Generate report
  ↓
Output: documents/notes/fact-check-report.md
        documents/notes/fact-check-raw-[timestamp].md
```

### Phase 2: Apply Corrections

```
User: "/apply-fact-check"
  ↓
Skill: apply-fact-check/SKILL.md executes
  ↓
Step 1: Load report + original article
  ↓
Step 2: Research updated data for each flagged claim
  ↓
Step 3: Apply corrections to article content
  ↓
Step 4: Generate detailed changelog
  ↓
Step 5: Save outputs
  ↓
Output: documents/notes/fact-check-changelog.md
        documents/notes/fact-check-updated-article.md
        outputs/sessions/fact-check-updated-article.md
```

## MCP Integration

This project uses the **AirOps MCP** (`https://app.airops.com/mcp`) for knowledge base retrieval. The MCP connection enables:

- **Semantic search** across the brand's knowledge base to trace where article data originated
- **Source verification** by matching claims against internal documentation
- **Freshness checks** to see if KB data has been updated since the article was written

If the AirOps MCP is not connected, both skills gracefully degrade to web-only verification.

### MCP Configuration

```json
{
    "mcpServers": {
        "airops": {
            "url": "https://app.airops.com/mcp"
        }
    }
}
```

## Structure

```
.claude/
├── agents/                          # Specialized sub-agents
│   ├── researcher.md                    → Gathers information
│   └── writer.md                        → Drafts content
└── skills/                          # Skills and slash commands
    ├── fact-check/                      → Phase 1: Analyze & report
    │   └── SKILL.md
    ├── apply-fact-check/                → Phase 2: Correct & output
    │   └── SKILL.md
    ├── save/                            → Saves session to outputs/
    ├── note-taking/                     → How to format notes
    └── knowledge-search/                → How to search documents/

documents/                           # Knowledge base (Claude can read & write)
├── notes/
│   ├── fact-check-report.md             # Verification report (Phase 1 output)
│   ├── fact-check-raw-[timestamp].md    # Scraped article content (Phase 1 output)
│   ├── fact-check-changelog.md          # Edit log (Phase 2 output)
│   └── fact-check-updated-article.md    # Corrected article (Phase 2 output)
├── templates/                       # Reusable formats
└── style-guide.md                   # Writing standards

outputs/                             # Session artifacts from /save
└── sessions/
    └── fact-check-updated-article.md    # Delivery copy (Phase 2 output)
```

## Workflow Example

A typical session looks like this:

```
> /fact-check https://www.servicetitan.com/licensing/hvac/iowa

🔍 Scraping article...
📋 Extracted 23 verifiable claims
📚 Cross-referencing AirOps Knowledge Base...
🌐 Running deep web research on 18 claims...
⚠️  Found 4 outdated, 1 incorrect, 2 unverifiable claims

📄 Report saved: documents/notes/fact-check-report.md

> /apply-fact-check

📖 Loading report... 7 claims to correct
🔬 Researching updated data...
✏️  Applying corrections...

✅ Corrections applied!
📝 7 edits made:
   - 4 outdated claims updated
   - 1 incorrect claim corrected
   - 0 hallucinated items removed
   - 2 unverifiable claims hedged (review recommended)

📄 Files saved:
   - documents/notes/fact-check-changelog.md
   - documents/notes/fact-check-updated-article.md
   - outputs/sessions/fact-check-updated-article.md
```

## Classification System

Both skills use a consistent flag system:

| Flag | Meaning | Action |
|------|---------|--------|
| ✅ Verified | Confirmed by authoritative source | No change |
| ⚠️ Outdated | Was true, now changed | Update with current data |
| ❌ Incorrect | Factually wrong | Correct with accurate data |
| 🔍 Unverifiable | No source found | Hedge or remove |
| 🤖 Likely Hallucinated | Appears fabricated | Remove or replace with real data |
| ⚡ Stale KB Source | KB data may be outdated | Refresh from current sources |
