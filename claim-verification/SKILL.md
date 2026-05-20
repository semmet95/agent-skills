---
name: claim-verification
description: >
  Given a link to a media post containing a falsifiable claim, extract the core claim, use web search tools to find up to two high-quality external documents that directly support or refute it, and output the URLs with their verdict.
tags: [news, fact-checking, falsifiability, web-search, evidence-retrieval]
---

# claim-verification

## Goal

Take a **falsifiable claim** made by a news or media outlet as input (via its URL), understand the claim from that page, then use web search tools to find up to **two external documents** that **directly and conclusively** support or refute the claim.

Output **only a JSON array** with URLs and whether each supports the claim.

> Because LLMs can hallucinate or misrepresent sources, this skill focuses on retrieving and returning **real, verifiable URLs**, not generating narrative fact-checks.

---

## Inputs

Required:

- `claim_url` (string)  
  Direct link to the media article or post that contains the claim to be checked.

Optional:

- `claim_hint` (string)  
  User-provided short description of the claim. Use it only as a hint; the final understanding must be based on the actual page content.
- `language` (string)  
  Language code of the page (e.g., `"en"`). If omitted, assume English unless the fetched page clearly indicates otherwise.

---

## Outputs

Output **only a JSON array of objects** (0-2 items), where each object has:

- `url`: string - the document URL
- `supports_claim`: boolean - `true` if the document supports the claim, `false` if it refutes it

```json
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
```

**Constraints:**
- Maximum 2 objects
- Each URL must correspond to a real, retrieved document that directly addresses the claim
- No text, explanation, or code fences - only the JSON array

---

## Definitions

- **Falsifiable claim**: A concrete assertion about reality that can be proven true or false by evidence (e.g., "X happened on date Y", "Organization Z reported N cases").

- **Conclusive document**: A source that explicitly and directly confirms or contradicts the claim's core factual content (e.g., official statistics, press releases, fact-checkers).

> All evidence must come from retrieved documents, not internal knowledge.

---

## Step-by-step instructions

### 1. Retrieve and understand the claim

1. Use the fetch tool to access `claim_url`. If unavailable, use web search with the URL or article title + outlet name.
2. Identify the **main falsifiable claim** from the headline and lead paragraphs.
3. Rewrite this claim internally as a single precise sentence (do not output).
4. Check that the claim is **falsifiable**: concrete (who/what/when/where/how much) and testable using external evidence.

**Claim handling:**
- Strip away framing, adjectives, and editorial language
- If compound, isolate the main testable statement
- If too vague to verify, be prepared to return `[]`

### 2. Design targeted search queries

5. Break the claim into: entities, actions/events, timeframes, quantities.
6. Construct **2–4 focused web search queries** combining core entities with keywords like `"official statement"`, `"press release"`, `"report"`, `"fact check"`, `"data"`, `"statistics"`.

**Example query patterns:**
- `"[exact statistic or event]" site:gov`
- `"[official name]" press release [date]`
- `"[claim subject]" fact check OR investigation`

### 3. Gather candidate sources

7. Use web search to retrieve candidate pages.
8. For each promising result, load the page and check if it directly addresses the claim (same core event/data/decision).
9. Discard pages that only loosely mention the topic, or are forum threads, low-quality blogs, or user-generated content.

### 4. Select up to two conclusive documents

10. Identify sources that most directly speak to the claim.

**Source preference order:**
1. Official/public institutional sources (government, courts, regulators, central banks, statistical agencies, official company press centers)
2. Major reputable news organizations and established fact-checkers
3. High-quality specialized outlets or expert organizations

**Selection criteria:**
- Clearly confirm or contradict the claim
- Independent of each other
- More authoritative than the original media post
- Directly relevant (not passing mentions)

**Avoid:**
- Duplicate reports of the same underlying source
- Circular citations
- Opinion pieces

If no source directly addresses the claim, prepare to output `[]`.

### 5. Avoid hallucinations

11. Never fabricate URLs, titles, or publishers. Every URL must correspond to a page actually retrieved via tools.
12. If unsure about relevance or authenticity, do not output the URL. Better to output fewer items or `[]` than misleading sources.

### 6. Produce final output

13. Output the JSON array containing 0-2 objects, each with `url` and `supports_claim` fields.

**Examples:**

Two documents (one supports, one refutes):
```json
[
  {
    "url": "https://official.gov/statistics/report-1234",
    "supports_claim": true
  },
  {
    "url": "https://factcheck.org/article/5678",
    "supports_claim": false
  }
]
```

One document:
```json
[
  {
    "url": "https://example.com/only-document",
    "supports_claim": true
  }
]
```

No suitable evidence:
```json
[]
```

Your final answer must be **only** the JSON array.
