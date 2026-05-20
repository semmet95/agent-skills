---
name: claim-verification
description: >
  Given a link to a media post containing a falsifiable claim, extract the core claim, use web search tools to find up to two high-quality external documents that directly support or refute it, and output the URLs with their verdict.
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
- `url`: string - the document URL
- `supports_claim`: boolean - `true` if supports, `false` if refutes

[
  {
    "url": "https://example.org/document-1",
    "supports_claim": true
  },
  {
    "url": "https://example.com/document-2",
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

## Step-by-step instructions

### 1. Retrieve and understand the claim

1. Use fetch tool to access `claim_url`. If unavailable, use web search with URL or article title.
2. Identify the **main falsifiable claim** from headline and lead paragraphs.
3. Verify claim is **falsifiable**: concrete (who/what/when/where/how much) and testable.
4. Strip away framing, adjectives, and editorial language. If compound, isolate the main testable statement.

### 2. Design targeted search queries

5. Break claim into: entities, actions/events, timeframes, quantities.
6. Construct **2–4 queries** combining core entities with keywords: `"official statement"`, `"press release"`, `"report"`, `"fact check"`, `"data"`, `"statistics"`.

**Example patterns:**
- `"[exact statistic]" site:gov`
- `"[official name]" press release [date]`
- `"[subject]" fact check OR investigation`

### 3. Gather candidate sources

7. Use web search to retrieve candidate pages.
8. For each promising result, verify it directly addresses the claim (same core event/data/decision).
9. Discard pages that loosely mention the topic, or are forums, low-quality blogs, or user-generated content.

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

**Avoid:** duplicate reports, circular citations, opinion pieces

If no source directly addresses the claim, output `[]`.

### 5. Produce final output

11. Never fabricate URLs. Every URL must correspond to a page actually retrieved.
12. If unsure about relevance or authenticity, omit the URL.
13. Output JSON array with 0-2 objects.
14. Do not enclose the json string in code fences.

**Examples:**

[
  {
    "url": "https://official.gov/report-1234",
    "supports_claim": true
  }
]

[]

Your final answer must be **only** the JSON array.
