---
name: claim-verification
description: >
  Given a link to a media post containing a falsifiable claim, extract the core claim, use time-aware web search to find up to two high-quality external documents that directly support or refute it, and output only the URLs of those documents.
tags: [news, fact-checking, falsifiability, web-search, evidence-retrieval]
---

# claim-verification

## Goal

Take a **falsifiable claim** made by a news or media outlet as input (via its URL), then use web search to find up to **two external documents** that **directly and conclusively** support or refute the claim.

> This skill focuses on retrieving **real, verifiable URLs**, not generating narrative fact-checks.

---

## Inputs

Required:

- `claim_url` (string) - Direct link to the media article or post containing the claim

Optional:

- `claim_hint` (string) - User-provided claim description (use only as a hint)
- `language` (string) - Language code (e.g., `"en"`). Default: English

---

## Outputs

Output **only a JSON array** (0-2 objects), where each object has:
- `uri`: string - the document URL
- `supports_claim`: boolean - `true` if supports, `false` if refutes

[
  {
    "uri": "https://example.org/document-1",
    "supports_claim": true
  },
  {
    "uri": "https://example.com/document-2",
    "supports_claim": false
  }
]

No text, explanation, or code fences - only the JSON array.

---

## Definitions

- **Falsifiable claim**: Concrete assertion that can be proven true or false by evidence (e.g., "X happened on date Y", "Organization Z reported N cases")
- **Conclusive document**: Source that explicitly confirms or contradicts the claim's core factual content

> All evidence must come from retrieved documents, not internal knowledge.

---

## Temporal relevance rules

When evaluating candidate documents, you must consider **time** alongside content relevance:

1. **Determine the claim timeframe** from the media article:
   - Explicit year (e.g., “in 2025”).
   - Relative references (e.g., “this year”, “last year”, “over the past 12 months”), which you should interpret relative to the article’s publication date.
   - If multiple periods are mentioned, focus on the one that is central to the claim (typically the one in the headline or lede).

2. **Preferred evidence window**:
   - For **point-in-time claims** (e.g., “wildfires increased in 2025”):
     - Prefer sources published **during the claimed period** or **soon after**, typically within **the same calendar year or the year immediately following**.
   - For **period-based claims** (e.g., “over the past decade”):
     - Prefer sources whose data and publication dates **cover or extend through the end of that period**.
   - For **timeless institutional facts** (e.g., “WHO is a UN agency”):
     - Temporal constraints are less strict; any reliable, recent source is acceptable.

3. **Avoid obsolete or misaligned evidence**:
   - Do **not** treat sources as conclusive evidence if:
     - They stop **before** the relevant period (e.g., a 2022 report for a claim about 2025 numbers).
     - They only discuss **earlier years** without explicitly projecting or confirming the claimed period.
   - Earlier sources can still be used internally for context, but should generally **not** be selected as one of the final 1–2 “conclusive” URLs if they do not address the claim period.

4. **Exceptions**:
   - A document that clearly states **multi-year data including the claim year** (e.g., a 2026 report summarizing data from 2020–2025) is acceptable as evidence for a claim about 2025.
   - A document that **formally corrects or retracts** an earlier claim can be used even if published later than the claim period.

If you cannot find a temporally appropriate document that directly addresses the claim, prefer to return **`[]`** rather than relying on outdated or misaligned evidence.

---

## Step-by-step instructions

### 1. Retrieve and understand the claim

1. Use fetch tool to access `claim_url`. If unavailable, use web search with URL or article title.
2. Identify the **main falsifiable claim** from headline and lead paragraphs.
3. Extract the **claim's date/timeframe** (when the event allegedly occurred or the data was reported).
4. Verify claim is **falsifiable**: concrete (who/what/when/where/how much) and testable.
5. Strip away framing, adjectives, and editorial language. If compound, isolate the main testable statement.

### 2. Design targeted search queries

6. Break claim into: entities, actions/events, timeframes, quantities.
7. **Include the claim's date/timeframe** in search queries to ensure temporal relevance. Use date filters or year-specific terms.
8. Construct **2–4 queries** combining core entities and dates with keywords: `"official statement"`, `"press release"`, `"report"`, `"fact check"`.

**Example patterns:**
- `"[exact statistic]" site:gov [year]`
- `"[official name]" press release [specific date]`
- `"[subject]" [year] fact check OR investigation`

### 3. Gather candidate sources

9. Use web search to retrieve candidate pages.
10. For each promising result, verify:
    - It directly addresses the claim (same core event/data/decision)
    - **Publication date matches the claim's timeframe** (e.g., for a 2025 claim, documents from 2024 or earlier are likely irrelevant unless providing historical context)
11. Discard pages that:
    - Loosely mention the topic without addressing the specific claim
    - Are from wrong time periods (e.g., old reports about similar but different events)
    - Are forums, low-quality blogs, or user-generated content

### 4. Select up to two conclusive documents

10. Identify sources that most directly speak to the claim.

**Source preference:**
1. Official/public institutional sources (government, courts, regulators, statistical agencies, official company press centers)
2. Major reputable news organizations and established fact-checkers
3. High-quality specialized outlets

**Select sources that:**
- Clearly confirm or contradict the claim
- Are independent of each other
- Are more authoritative than the original media post
- Directly address the claim (not passing mentions)
- **Are temporally relevant** (published around the same time as the claim's event, not years apart)

**Avoid:**
- Duplicate reports, circular citations, opinion pieces
- **Documents from different time periods** (e.g., 2024 fire reports as proof for 2025 fire claims)

If no source directly addresses the claim, output `[]`.

### 5. Produce final output

11. Never fabricate URLs. Every URL must correspond to a page actually retrieved.
12. If unsure about relevance or authenticity, omit the URL.
13. Output JSON array with 0-2 objects.
14. Do not enclose the json string in code fences.

**Examples:**

[
  {
    "uri": "https://official.gov/report-1234",
    "supports_claim": true
  }
]

[]

Your final answer must be **only** the JSON array.
