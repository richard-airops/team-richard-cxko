---
name: fact-check
description: Fact-check an article by scraping the URL, extracting all claims and data points, cross-referencing against the AirOps knowledge base and live web research, then outputting a verification report with corrections.
argument-hint: [url]
disable-model-invocation: false
allowed-tools: Read, Grep, Glob, Bash(curl:*), Bash(echo:*), Bash(mkdir:*), Bash(cat:*), Bash(jq:*), mcp__airops__search_knowledge_base, mcp__airops__list_workspaces, mcp__airops__execute_workflow, WebSearch, WebFetch
---

# Fact-Check Skill

Fact-check an article at the provided URL by systematically verifying every claim, data point, and statistic against internal knowledge base sources and live web research. Produces a structured report flagging anything unverifiable, outdated, or hallucinated — with suggested corrections.

## Input

The user provides a single URL as `$ARGUMENTS`. Example:

```
/fact-check https://www.servicetitan.com/licensing/hvac/iowa
```

## Execution Pipeline

Follow these 6 steps **in order**. Do not skip steps. Log progress to the user after each phase.

---

### Step 1: Scrape the URL for Content

Fetch the full page content from the provided URL.

```bash
curl -sL -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" "$ARGUMENTS" | sed 's/<script[^>]*>.*<\/script>//g; s/<style[^>]*>.*<\/style>//g; s/<[^>]*>//g; s/&nbsp;/ /g; s/&amp;/\&/g; s/&lt;/</g; s/&gt;/>/g' | sed '/^$/d' | head -500
```

- If curl fails, try `WebFetch` as a fallback.
- Strip HTML tags, scripts, and styles to get clean text.
- Save the raw extracted content to `documents/notes/fact-check-raw-$TIMESTAMP.md` for reference.

**Output of this step:** Clean article text ready for claim extraction.

---

### Step 2: Identify All Claims, Data Points & Facts

Analyze the scraped content and extract **every verifiable element** into a structured inventory. Categorize each item:

| Category | Examples |
|----------|---------|
| **Statistics** | Percentages, dollar amounts, counts, measurements |
| **Regulatory/Legal** | License requirements, laws, regulations, compliance standards |
| **Dates & Timelines** | "As of 2024", "effective January 1", deadlines |
| **Named Entities** | Organizations, agencies, people, programs referenced |
| **Process Claims** | "You must apply through X", "the process takes Y days" |
| **Pricing/Cost** | Fees, costs, price ranges |
| **Geographic** | State-specific rules, jurisdictional claims |
| **Comparative** | "Better than X", "leading provider", rankings |

For each claim, record:
- **Claim ID** (sequential: C1, C2, C3...)
- **Exact quote** from the article
- **Category** from the table above
- **Confidence level** (how specific/verifiable is this claim?)

Save the full inventory to a working file for reference.

---

### Step 3: Knowledge Base Cross-Reference (AirOps MCP)

For each claim or logical group of related claims, query the AirOps knowledge base via the MCP connection to find **internal source material** that may have been used to generate or inform this content.

Use the `mcp__airops__search_knowledge_base` tool to:

1. **Search by topic keywords** — Extract the core subject of each claim and search for matching knowledge base entries.
2. **Search by exact phrases** — For specific statistics or quotes, search the exact text to find if it originated from a knowledge base document.
3. **Search by entity names** — For any organizations, agencies, or programs mentioned, search to find source documents.

For each claim, record:
- **KB Match Found?** (Yes/No)
- **KB Source Document** (document name + chunk if available)
- **KB Match Quality** (Exact match / Partial match / Topically related / No match)
- **KB Data Freshness** (date of the KB document if available)
- **Discrepancies** (any differences between the article claim and KB source)

> **Important:** If no knowledge base is configured or accessible, note this in the report and proceed to Step 4. The knowledge base step helps trace *where* data came from internally — it does not replace web verification.

---

### Step 4: Deep Web Research Verification

For **every claim** that meets any of these criteria, perform live web research:
- No KB match was found
- KB match is partial or outdated
- Claim involves a statistic, regulation, date, or fee amount
- Claim is high-stakes (legal/regulatory/financial)

Research approach per claim:

1. **Primary source search** — Find the original authoritative source (government website, official database, regulatory body, peer-reviewed publication).
2. **Recency check** — Verify the information is current. Look for the most recent version of regulations, fee schedules, requirements, etc.
3. **Cross-reference** — Check at least 2 independent sources for critical claims.

Use `WebSearch` for targeted queries and `WebFetch` to pull full pages from authoritative sources when needed.

For each researched claim, record:
- **Verification Status**: ✅ Verified | ⚠️ Outdated | ❌ Incorrect | 🔍 Unverifiable
- **Source(s)** with URLs
- **Current correct information** (if different from article)
- **Date of verification source**

---

### Step 5: Identify Outdated, Incorrect & Hallucinated Content

Synthesize findings from Steps 3 and 4. Flag every issue found:

**Classification system:**

| Flag | Meaning | Severity |
|------|---------|----------|
| ❌ **INCORRECT** | Claim is factually wrong based on authoritative sources | 🔴 Critical |
| ⚠️ **OUTDATED** | Claim was once true but information has since changed | 🟠 High |
| 🔍 **UNVERIFIABLE** | No authoritative source found to confirm or deny | 🟡 Medium |
| 🤖 **LIKELY HALLUCINATED** | Claim appears fabricated — no source exists and details are suspiciously specific | 🔴 Critical |
| ⚡ **STALE KB SOURCE** | KB has the data but it may be outdated vs. current reality | 🟡 Medium |
| ✅ **VERIFIED** | Claim confirmed by authoritative source(s) | 🟢 OK |

For each flagged issue, prepare:
- The original claim (exact quote)
- Why it's flagged
- The correct/current information
- Source URL(s) for the correction
- Suggested replacement text

---

### Step 6: Generate the Fact-Check Report

Create the final report and save it to `documents/notes/fact-check-report.md`.

Use the following template:

```markdown
# Fact-Check Report

**URL:** [article URL]
**Date Checked:** [today's date]
**Total Claims Analyzed:** [count]
**Verification Summary:**

| Status | Count |
|--------|-------|
| ✅ Verified | X |
| ⚠️ Outdated | X |
| ❌ Incorrect | X |
| 🔍 Unverifiable | X |
| 🤖 Likely Hallucinated | X |
| ⚡ Stale KB Source | X |

## Overall Risk Assessment

[One paragraph summary: Is this article safe to publish as-is? What's the severity of issues found? Recommended action — publish, revise, or hold for major rewrite.]

## Critical Issues (Immediate Fixes Required)

### [C#] — [Brief description]
- **Original claim:** "[exact quote]"
- **Issue:** [what's wrong]
- **Correct information:** [current/correct data]
- **Source:** [URL]
- **Suggested fix:** "[replacement text]"

---

## Warnings (Should Fix Before Publishing)

### [C#] — [Brief description]
- **Original claim:** "[exact quote]"
- **Issue:** [what's wrong]
- **Correct information:** [current/correct data]
- **Source:** [URL]
- **Suggested fix:** "[replacement text]"

---

## Unverifiable Claims (Review Recommended)

### [C#] — [Brief description]
- **Original claim:** "[exact quote]"
- **Issue:** Could not find authoritative source to verify
- **Recommendation:** [remove, add citation, or rephrase as opinion]

---

## Verified Claims (No Action Needed)

| Claim ID | Summary | Source |
|----------|---------|--------|
| C# | [brief] | [source URL] |

---

## Knowledge Base Traceability

| Claim ID | KB Match | KB Document | Match Quality | Notes |
|----------|----------|-------------|---------------|-------|
| C# | Yes/No | [doc name] | Exact/Partial/None | [notes] |

---

## Appendix: Full Claim Inventory

[Include the complete claim inventory from Step 2 with final verification status for each]
```

---

## Important Notes

- **Always scrape first.** Do not assume you know the article content. Fetch it fresh.
- **Be exhaustive.** Every verifiable claim should be checked. Do not skip claims because they "seem right."
- **Prioritize authoritative sources.** Government sites (.gov), official regulatory bodies, and primary sources over blog posts or secondary reporting.
- **Date sensitivity matters.** Regulations, fees, and requirements change. A claim that was correct in 2023 may be wrong in 2025. Always check for the most current version.
- **Knowledge base is for traceability, not truth.** The KB tells you where the data *came from* internally. The web research tells you if it's still *correct*. Both are needed.
- **If the AirOps MCP is not connected**, skip Step 3 gracefully and note it in the report. The skill should still work for web-only verification.
- **Save the report** to `documents/notes/fact-check-report.md` and also display a summary to the user in the conversation.

## Output

The skill produces two things:
1. **`documents/notes/fact-check-report.md`** — The full detailed report saved to the knowledge base
2. **Conversation summary** — A brief overview of findings shown directly to the user with the critical issues highlighted

## File Structure

```
documents/
└── notes/
    ├── fact-check-raw-[timestamp].md    # Raw scraped content
    └── fact-check-report.md             # Final verification report
```

## Error Handling

| Issue | Fallback |
|-------|----------|
| URL returns 403/404 | Try WebFetch. If still fails, ask user for alternate URL or pasted content. |
| AirOps MCP not connected | Skip KB step, note in report, proceed with web-only verification. |
| No claims extractable | Report that the page has no verifiable claims (e.g., purely navigational page). |
| Rate limited on web search | Space out requests. Prioritize critical/regulatory claims first. |
