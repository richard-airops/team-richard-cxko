---
name: apply-fact-check
description: Read the saved fact-check report, research updated data for all flagged claims, apply corrections to the original article content, and output a changelog and corrected article.
argument-hint: [optional-report-path]
disable-model-invocation: false
allowed-tools: Read, Grep, Glob, Bash(curl:*), Bash(echo:*), Bash(mkdir:*), Bash(cat:*), Bash(sed:*), Bash(diff:*), Bash(cp:*), Bash(date:*), Bash(jq:*), mcp__airops__search_knowledge_base, mcp__airops__list_workspaces, mcp__airops__execute_workflow, WebSearch, WebFetch
---

# Apply Fact-Check Skill

Takes a previously saved fact-check report (from `/fact-check`) and applies corrections to the original article. Researches current data for every flagged claim, rewrites the affected content, and produces a clean updated article alongside a detailed changelog of every edit made.

## Source Quality Policy

**This skill enforces strict source quality standards.** The following source types are **prohibited** and must never be used as authoritative references for corrections:

### Blocked Source Categories

| Category | Examples | Why Blocked |
|----------|----------|-------------|
| Social media platforms | Twitter/X, Facebook, Instagram, Threads, TikTok, Bluesky, Mastodon | User-generated, unvetted, easily fabricated |
| Short-form video platforms | TikTok, YouTube Shorts, Instagram Reels | No editorial review, often misleading |
| Discussion forums | Reddit, Quora, Stack Exchange (non-technical), Yahoo Answers | Crowdsourced opinion, not authoritative |
| User-generated wikis | Wikipedia (as a primary source), Fandom wikis | Editable by anyone, use their cited sources instead |
| Personal blogs & newsletters | Medium, Substack, personal sites | No editorial oversight |
| AI-generated content hubs | Sites that aggregate LLM-generated answers | Circular sourcing risk |

### Blocked Domains List

When fetching sources, **skip any URL** matching these domains:

```
twitter.com, x.com, facebook.com, instagram.com, threads.net,
tiktok.com, reddit.com, quora.com, medium.com, substack.com,
linkedin.com/posts, linkedin.com/pulse, tumblr.com, pinterest.com,
bluesky.social, mastodon.social, mastodon.online, bsky.app,
news.ycombinator.com, stackexchange.com (non-stackoverflow.com),
youtube.com/shorts, t.me, discord.com, discord.gg
```

**Exception:** YouTube long-form videos from **official government or organizational channels** are acceptable as supporting (not primary) sources. LinkedIn company pages (not posts/pulse articles) may be used to verify company-specific claims only.

### Preferred Source Hierarchy

When researching corrections, prefer sources in this order:

1. **Primary government sources** — .gov, .gov.uk, .gc.ca, official regulatory body sites
2. **Official organizational sites** — the authoritative body for the topic (e.g., professional licensing boards, standards bodies)
3. **Peer-reviewed publications** — academic journals, published research
4. **Established news organizations** — AP, Reuters, NYT, WSJ, BBC, etc. (with editorial standards)
5. **Industry-recognized reference sites** — established trade publications, professional associations
6. **AirOps Knowledge Base** — internal KB entries (with freshness check)

If the **only** source available for a correction is from a blocked category, **do not apply the correction**. Instead, hedge the claim and flag it in the changelog as:

> ⚠️ **No authoritative source found.** Only social media / user-generated references available. Claim hedged pending manual verification.

## Input

This skill consumes the **saved outputs** from a prior `/fact-check` run:

- **Fact-check report:** `documents/notes/fact-check-report.md`
- **Raw article content:** `documents/notes/fact-check-raw-*.md`

If `$ARGUMENTS` is provided, use it as the path to the report. Otherwise, default to `documents/notes/fact-check-report.md`.

```
/apply-fact-check
/apply-fact-check documents/notes/fact-check-report.md
```

## Prerequisites

This skill **requires** a completed `/fact-check` run. If neither the report nor the raw article content exist, instruct the user:

> No fact-check report found. Run `/fact-check [url]` first, then come back and run `/apply-fact-check` to apply the corrections.

## Execution Pipeline

Follow these 5 steps **in order**. Log progress to the user after each phase.

---

### Step 1: Load the Fact-Check Report & Original Article

1. Read the fact-check report from `documents/notes/fact-check-report.md` (or the path provided in `$ARGUMENTS`).
2. Find and read the most recent raw article file matching `documents/notes/fact-check-raw-*.md`.
3. Parse the report to extract:
   - **All flagged claims** — anything marked ❌ INCORRECT, ⚠️ OUTDATED, 🤖 LIKELY HALLUCINATED, or ⚡ STALE KB SOURCE.
   - **Their suggested fixes** from the report.
   - **The original article URL** (from the report header).
   - **Unverifiable claims** (🔍) — these need a decision, not a data swap.

4. Build a **corrections queue** — a structured list of every claim that needs updating:

```
Corrections Queue:
- C3 | ⚠️ OUTDATED | "License fee is $150" → Suggested: "$175 as of 2025"
- C7 | ❌ INCORRECT | "Apply through the DOL" → Suggested: "Apply through the DPH"
- C12 | 🤖 HALLUCINATED | "Processing takes 3-5 business days" → Suggested: Remove or verify
...
```

**Output of this step:** Loaded report, loaded article, corrections queue ready.

---

### Step 2: Research Updated Data for Each Flagged Claim

For every claim in the corrections queue, perform targeted research to get **current, authoritative data** to replace the flagged content.

**Do not blindly use the "suggested fix" from the fact-check report.** The suggestions are starting points. This step verifies and improves them with fresh research.

**⚠️ ENFORCE SOURCE QUALITY POLICY:** Before using any source found during research, check the URL against the blocked domains list above. If a search result comes from a blocked source, **discard it** and continue searching for an authoritative alternative.

For each flagged claim:

1. **Re-search the specific data point** — Use `WebSearch` to find the most current authoritative source. When evaluating results, **skip any result from a blocked domain** (social media, forums, user-generated content). Prefer .gov, .edu, .org, and established news domains.
2. **Fetch primary sources** — Use `WebFetch` on government sites, official databases, or regulatory bodies to get exact current figures, requirements, or processes. **Never fetch from blocked domains** — if a search result points to a social media post as the only source, treat the claim as unverifiable.
3. **Cross-reference with KB** — Use `mcp__airops__search_knowledge_base` to check if the AirOps knowledge base has been updated since the original fact-check. If the KB has fresher data, prefer it.
4. **Validate the source before accepting** — Every correction must have:
   - The **exact new text** to insert
   - The **authoritative source URL** backing it (must NOT be from a blocked domain)
   - The **source category** from the preferred hierarchy (e.g., "Primary government source", "Established news org")
   - The **date** the source was checked

**Source validation checklist (apply to every source before using it):**
- [ ] URL is not on the blocked domains list
- [ ] Source has editorial oversight or official authority
- [ ] Content is dated or has a clear publication/update date
- [ ] Source is the originator of the data (not citing a social media post)
- [ ] If a news article, it cites primary sources you can verify

**For 🔍 UNVERIFIABLE claims:**
- If research now finds an **authoritative** source → upgrade to ✅ VERIFIED or apply correction
- If only social media sources exist → remains unverifiable. Hedge and flag.
- If still unverifiable → flag for the user to decide: keep, remove, or rephrase as hedged language (e.g., "approximately", "typically", "may vary")

**For 🤖 LIKELY HALLUCINATED claims:**
- If research finds the real data **from an authoritative source** → replace with verified information
- If only social media or forum sources confirm the data → treat as still unverified. Remove or hedge.
- If no real data exists → remove the claim entirely and note the removal

Save the researched corrections to a working file for reference.

---

### Step 3: Apply Corrections to the Article

Take the original raw article content and apply each correction **surgically**:

1. **Work through the corrections queue in document order** (top to bottom of the article) to avoid offset issues.
2. For each correction:
   - Locate the exact original text in the article
   - Replace it with the researched updated text
   - Ensure the surrounding context still reads naturally — adjust transition sentences if needed
   - Do **not** change any content that was marked ✅ VERIFIED
3. **For removed claims** (hallucinated content with no replacement):
   - Remove the sentence/paragraph
   - Smooth over the gap so the article flows naturally
4. **For unverifiable claims the user hasn't decided on:**
   - Add hedging language (e.g., "This may vary by jurisdiction" or "Contact [authority] for current requirements")
   - Flag these in the changelog as "auto-hedged — review recommended"

**Important rules:**
- Preserve the article's tone, voice, and structure
- Do not rewrite sections that don't need changes
- Keep formatting (headers, lists, links) intact
- If a correction affects a heading or subheading, update it too
- If multiple claims are in the same paragraph, apply all corrections to that paragraph together to maintain coherence

---

### Step 4: Generate the Changelog

Create a detailed changelog documenting every edit made. Save to `documents/notes/fact-check-changelog.md`.

Use this template:

```markdown
# Fact-Check Corrections Changelog

**Original URL:** [article URL]
**Report Used:** [path to fact-check report]
**Corrections Applied:** [today's date]
**Total Edits:** [count]
**Source Policy:** Social media and user-generated content excluded

## Summary

| Edit Type | Count |
|-----------|-------|
| ⚠️ Outdated data updated | X |
| ❌ Incorrect data corrected | X |
| 🤖 Hallucinated content removed | X |
| ⚡ Stale KB data refreshed | X |
| 🔍 Unverifiable claims hedged | X |
| 🚫 Corrections blocked (no authoritative source) | X |

---

## Source Quality Report

All sources used for corrections were validated against the source quality policy.

| Source Category | Count |
|----------------|-------|
| Primary government source (.gov) | X |
| Official organizational site | X |
| Peer-reviewed publication | X |
| Established news organization | X |
| Industry reference site | X |
| AirOps Knowledge Base | X |

**Blocked sources encountered and discarded:** X
[List any search results that were found but rejected due to source policy, with the blocked domain noted]

---

## Detailed Changes

### Edit 1 — [C#] [Category]
- **Type:** [OUTDATED / INCORRECT / HALLUCINATED / STALE KB / UNVERIFIABLE]
- **Original text:**
  > "[exact original text from article]"
- **Updated text:**
  > "[exact new text as applied]"
- **Reason:** [why this was changed]
- **Source:** [URL of authoritative source]
- **Source category:** [e.g., "Primary government source"]
- **Confidence:** [High / Medium — based on source quality]

---

### Edit 2 — [C#] [Category]
...

---

## Removals

[List any content that was removed entirely, with the original text and reason for removal]

## Corrections Not Applied (No Authoritative Source)

[List any flagged claims where the only available sources were from blocked categories. Include the original claim, what was found, and why it was rejected.]

## Auto-Hedged Claims (Review Recommended)

[List any unverifiable claims where hedging language was added automatically. The user should review these and decide whether to keep, strengthen, or remove them.]

## Unchanged Verified Claims

[Count of claims that were verified and left untouched]
```

---

### Step 5: Save Outputs

Save all outputs to the `documents/notes/` directory and the final updated article to `outputs/`:

1. **Changelog:** `documents/notes/fact-check-changelog.md`
2. **Updated article:** `documents/notes/fact-check-updated-article.md`
3. **Output copy for delivery:** `outputs/sessions/fact-check-updated-article.md`

The updated article should be saved as clean content with a small metadata header:

```markdown
---
source_url: [original URL]
fact_checked: [date of fact-check]
corrections_applied: [date of apply]
total_corrections: [count]
source_policy: "Social media and user-generated content excluded"
---

[Full corrected article content here]
```

Display a summary to the user:

```
✅ Corrections applied!

📝 X edits made:
   - X outdated claims updated
   - X incorrect claims corrected
   - X hallucinated items removed
   - X unverifiable claims hedged (review recommended)
   - X corrections blocked (no authoritative source available)

🛡️ Source quality:
   - X authoritative sources used
   - X social media / UGC sources encountered and discarded

📄 Files saved:
   - documents/notes/fact-check-changelog.md (detailed changelog)
   - documents/notes/fact-check-updated-article.md (corrected article)
   - outputs/sessions/fact-check-updated-article.md (delivery copy)

⚠️  X auto-hedged claims need your review — see changelog for details.
🚫 X corrections could not be applied due to source policy — see changelog.
```

---

## Important Notes

- **Never use social media as a source.** This is the top-priority rule. No correction should ever cite Twitter/X, Facebook, Reddit, TikTok, LinkedIn posts, or any other social media platform. If a social media post is the only source for a data point, the claim must be hedged, not corrected.
- **Never invent replacement data.** Every correction must be backed by an authoritative source found during Step 2. If you can't find current data for a flagged claim from a non-social-media source, hedge it — don't guess.
- **Preserve voice and tone.** The corrections should blend seamlessly into the existing article. Don't make the updated sections sound like a different author wrote them.
- **Changelog is the audit trail.** Every single edit must be logged with before/after text, a source, and the source category. This is non-negotiable — the user needs to be able to review and approve each change.
- **Don't touch verified content.** If the fact-check report marked something ✅ VERIFIED, leave it exactly as-is.
- **If the fact-check report is missing or incomplete**, tell the user and do not proceed. This skill depends on a prior `/fact-check` run.
- **If the AirOps MCP is not connected**, skip KB cross-referencing in Step 2 and rely on web research only. Note this in the changelog.
- **Wikipedia exception:** Wikipedia may be used to find cited primary sources (follow the footnotes), but Wikipedia itself should not be cited as the authoritative source. Cite the original source Wikipedia references instead.

## Output Files

```
documents/
└── notes/
    ├── fact-check-report.md              # From /fact-check (input — read only)
    ├── fact-check-raw-[timestamp].md     # From /fact-check (input — read only)
    ├── fact-check-changelog.md           # NEW: Detailed edit log
    └── fact-check-updated-article.md     # NEW: Corrected article

outputs/
└── sessions/
    └── fact-check-updated-article.md     # NEW: Delivery copy
```

## Error Handling

| Issue | Fallback |
|-------|----------|
| No fact-check report found | Instruct user to run `/fact-check [url]` first. Do not proceed. |
| Raw article file missing | Re-scrape the URL from the report header. |
| AirOps MCP not connected | Skip KB checks in Step 2, use web-only research. Note in changelog. |
| Can't find updated data for a claim | Hedge the claim with qualifying language. Flag in changelog as "needs manual review". |
| Only social media sources available | Do NOT use them. Hedge the claim and flag as "no authoritative source — social media only". |
| Original text not found in article (offset issue) | Use fuzzy matching to find the closest passage. Log a warning in the changelog. |
| News article cites social media as its source | Reject the news article too — find a source with independent verification. |
