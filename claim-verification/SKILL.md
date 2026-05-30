---
name: claim-verification
description: >
  Given a link to a media post containing a falsifiable claim, extract the core claim, use time-aware and semantically precise web search to find up to two high-quality independent external documents that directly support or refute it, and output only the URLs of those documents.
tags: [news, fact-checking, falsifiability, web-search, evidence-retrieval]
---

# claim-verification

## Goal

Take a **falsifiable claim** made by a news or media outlet as input (via its URL), then use web search to find up to **two external documents** that **directly and conclusively** support or refute the claim.

This skill focuses on retrieving **real, verifiable URLs**, not generating narrative fact-checks.

> Critical constraint 1: supporting or refuting proof must **never** come from the same source that made the claim. Use the claim source only to read and extract the claim itself.  
> Critical constraint 2: proof must be **about the same concrete event**, not just a similar topic (e.g., same village, same month/year, same type of incident, same parties involved).

---

## Inputs

Required:

- `claim_url` (string) – Direct link to the media article or post containing the claim

Optional:

- `claim_hint` (string) – User-provided claim description (use only as a hint)  
- `language` (string) – Language code (e.g., `"en"`). Default: English

---

## Outputs

Output **only a JSON array** (0–2 objects), where each object has:

- `uri`: string – the document URL  
- `supports_claim`: boolean – `true` if supports, `false` if refutes

Before any URL is included in the output, use the web search/browsing tool to visit that exact URL and verify that the page is reachable. Do not return URLs that fail to load, redirect to an unrelated page, require inaccessible authentication, or otherwise cannot be validated at output time.

Example:

```json
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
```

No text, explanation, or code fences – only the JSON array.

---

## Definitions

- **Falsifiable claim**  
  Concrete assertion that can be proven true or false by evidence (e.g., “X happened on date Y”, “Organization Z reported N cases”).

- **Conclusive document**  
  Source that explicitly confirms or contradicts the claim’s core factual content for the relevant **entities, location, timeframe, and quantities**.

- **Claim source**  
  The outlet, publication, author organization, domain, or account that originally made the claim.

> All evidence must come from retrieved documents, not internal knowledge.

---

## Source independence rule

The source that made the claim must not be used as supporting or refuting evidence.

1. Use `claim_url` only to read the article/post and extract the claim.  
2. Do **not** query the claim source about the claim after extraction.  
3. Do **not** search within the claim source’s domain, site, account, publication, or related syndicated pages for proof.  
4. Do **not** select any final evidence URL from the same source that made the claim, even if it appears to support, correct, retract, or refute the claim.  
5. If all available evidence comes from the claim source or circularly cites only the claim source, output `[]`.

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
     - They only discuss **earlier years or months** without explicitly projecting or confirming the claimed period.
       - For example, **do not** treat a May 2025 incident as proof for a claim specifically about June 2025 in the same village unless the document clearly covers both months as a single event or data series.
   - Earlier sources can still be used internally for context, but should generally **not** be selected as one of the final 1–2 “conclusive” URLs if they do not address the claim period.

4. **Exceptions**:
   - A document that clearly states **multi-year or multi-month data including the claim period** (e.g., a 2026 report summarizing data from 2020–2025, or a Jan–Dec 2025 annual report) is acceptable as evidence for a claim about 2025.
   - A document from an **independent source** that formally corrects or refutes an earlier claim can be used even if published later than the claim period, as long as it explicitly references that claim or event.

If you cannot find a temporally appropriate document that directly addresses the claim, prefer to return **`[]`** rather than relying on outdated or misaligned evidence.

---

## Semantic relevance rules

You must ensure that the selected documents are about **the same concrete event or data point**, not just a similar topic.

### 1. Extract a “claim signature”

From the claim article, identify and internally record the **claim signature** – a minimal set of elements that uniquely characterize the claim:

- **Entities**:
  - People (names, titles).
  - Organizations (companies, institutions, agencies).
  - Animals/objects explicitly involved (e.g., “panther”, “leopard”, “train”, “bridge”).

- **Location** (from most to least specific):
  - Village/neighborhood/ward name.
  - Town/city.
  - District/county.
  - State/province, country.

- **Timeframe**:
  - Exact date(s) if available (day, month, year).
  - Month/year or quarter/year if that is how the claim is framed.
  - Relative time descriptors resolved to a concrete period.

- **Event type**:
  - What happened? (e.g., “attacks”, “deaths”, “injuries”, “evacuations”, “arrests”, “law passed”).
  - Avoid conflating different event types (e.g., sightings vs attacks; injuries vs deaths).

- **Key quantitative details (if present)**:
  - Counts (e.g., “5 casualties”, “17 houses burned”).
  - Rates, percentages, amounts.

Also extract a short list of **keywords and proper nouns** from the claim (entities, locations, event terms) to use later for similarity checks.

### 2. Matching candidate documents against the claim signature

For each candidate document, check:

- **Entity match**:
  - Does it refer to the **same key entities** (people, organizations, animals) as the claim?
  - Generic mentions (e.g., “big cats” or “wild animals”) are weaker than explicit mentions (e.g., “panthers”).

- **Location match**:
  - Prefer documents that mention the **same specific location** (same village or town).
  - Documents that only match a broader region (same district/state but different village) should be treated as **weaker** and generally not used as conclusive evidence unless the claim itself is only at that broader level.

- **Timeframe match**:
  - Does the document clearly cover the **same date/month/year** as the claim?
  - If the claim is about **June**, a document about **May of the same year** is **not** conclusive evidence unless it clearly states that the same event or dataset spans both months.

- **Event type match**:
  - The type of event must match (e.g., “panther attacks causing casualties” is different from “panther sightings” or “panthers rescued by officials”).
  - If the document is about a **different type of incident** (e.g., property damage only vs deaths), treat it as non-conclusive unless the claim is at a more aggregate level.

- **Keyword/proper noun overlap**:
  - The document should contain **most of the critical keywords and proper nouns** from the claim signature (e.g., village name, key entities, exact month/year, “panther attacks” vs just “wildlife conflict”).
  - If only a small subset of these appears, or they appear in unrelated contexts, treat the document as **not directly related**.

If any of these core elements (location, timeframe, event type) clearly do **not** align, you must treat the document as **not directly about the claim**, even if it “sounds similar”.

---

## Step-by-step instructions

### 1. Retrieve and understand the claim

1. Use fetch tool to access `claim_url`. If unavailable, use web search with the URL or article title.  
2. Identify the **main falsifiable claim** from headline and lead paragraphs.  
3. Extract the **claim’s date/timeframe** (when the event allegedly occurred or the data was reported).  
4. Extract the **claim signature**:
   - Entities (people, organizations, animals) and their roles.
   - Location hierarchy (village → city → district → state → country).
   - Timeframe (date, month, year, period).
   - Event type (e.g., “attacks”, “casualties”, “evacuation”, “law passed”).
   - Key quantitative details (e.g., numbers of casualties, amounts).
   - A short list of **keywords and proper nouns** from the claim.
5. Verify the claim is **falsifiable**: concrete (who/what/when/where/how much) and testable.  
6. Strip away framing, adjectives, and editorial language. If the claim is compound, isolate the main testable statement.

### 2. Design targeted search queries

7. Break the claim into: entities, actions/events, timeframes, quantities, and locations.  
8. **Include the claim’s date/timeframe and key locations** in search queries to ensure temporal and geographic relevance. Use date filters or year-specific terms when available.  
9. Construct **2–4 queries** combining core entities, locations, and dates with keywords: `"official statement"`, `"press release"`, `"report"`, `"fact check"`, `"statistics"`, `"data"`.  
10. Exclude the claim source from evidence searches. If search syntax allows it, add a negative site/domain filter for the claim source.

Example patterns:

- `"[exact statistic]" site:gov [year]`  
- `"[village name]" "[district]" "[event type]" [month] [year]`  
- `"[official name]" press release [specific date or month year]`  
- `"[subject]" [year] report -site:claim-source.example`

### 3. Gather candidate sources

11. Use web search to retrieve candidate pages.  
12. For each promising result:
    - Verify that it directly addresses the **same claim** by checking against the claim signature:
      - Same or very closely matching **entities**.
      - **Same specific location** (or clearly documented coverage of that location).
      - **Same timeframe** (same date/month/year, or multi-period coverage that explicitly includes that period).
      - **Same event type** and similar quantitative scope.
      - High overlap with the claim’s extracted keywords and proper nouns.
    - Verify the **publication date** is consistent with the claim’s timeframe (see temporal rules).
    - Confirm it is not from the claim source or a page that merely republishes/syndicates the claim source.
13. Discard pages that:
    - Loosely mention the topic without matching the claim signature.
    - Are from wrong time periods (e.g., older incidents presented as proof for a newer one).
    - Are from the same source that made the claim.
    - Depend only on the claim source for the relevant factual assertion (circular reporting).
    - Are forums, low-quality blogs, or user-generated content, unless no better sources exist and they still match the claim signature well.

### 4. Select up to two conclusive documents

14. Identify sources that most directly and tightly match the **claim signature** and temporal constraints.

**Source preference:**

1. Official/public institutional sources (government, courts, regulators, statistical agencies, official company/NGO press centers).  
2. Major reputable news organizations and established fact-checkers.  
3. High-quality specialized outlets.

**Select sources that:**

- Clearly confirm or contradict the claim.  
- Are independent of each other.  
- Are independent of the original claim source.  
- Are more authoritative or directly data-based than the original media post.  
- Directly address the same event or dataset (not just similar events).  
- Are temporally relevant (cover the same date/month/year or explicitly include it in a clearly relevant period).

**Avoid:**

- Duplicate reports and circular citations.  
- Opinion pieces that only restate the claim without new evidence.  
- The original claim source, same-domain articles, same-account posts, syndicated copies, or later corrections/retractions from the same source.  
- Documents about **similar but different events**, such as:
  - Incidents in the same village but **in a different month or year**.  
  - Incidents in other villages or districts unless the claim itself is at that broader level.  
  - Different event types (e.g., sightings vs attacks; injuries vs deaths) unless the claim is clearly aggregate and the source clearly covers that dimension.

If no source directly addresses the claim with sufficient semantic and temporal alignment, output `[]`.

### 5. Decide supports_claim for each selected source

15. For each selected document:
    - Set `supports_claim` to `true` if the document’s content **clearly agrees with and corroborates** the claim’s core factual content (entities, location, timeframe, event type, and key numbers).  
    - Set `supports_claim` to `false` if the document **clearly contradicts** the claim (e.g., official correction, data that directly disproves the numbers or facts, or explicit fact-check labeling it false/misleading).  
    - If the document is mixed or ambiguous, you may still include it **only if**:
      - It directly references the claim or event, and  
      - It contains strong evidence one way or another, and  
      - You choose the `supports_claim` value that best reflects the overall direction of the evidence.

If you are not confident about how to set `supports_claim` for a document, omit that document instead of guessing.

### 6. Produce final output

16. Never fabricate URLs. Every URL must correspond to a page actually retrieved.  
17. Immediately before returning any selected URL, use the web search/browsing tool to visit the exact URL and confirm it is valid, reachable, and still points to the selected evidence document.  
18. If the URL is invalid, unreachable, gated in a way that prevents verification, or redirects to unrelated content, omit it from the output.  
19. If unsure about relevance, authenticity, independence from the claim source, or URL validity, omit the URL.  
20. Output a JSON array with 0–2 objects of the form:

```json
[
  {
    "uri": "https://official.gov/report-1234",
    "supports_claim": true
  },
  {
    "uri": "https://news.example.com/fact-check-5678",
    "supports_claim": false
  }
]
```

21. If no suitable evidence is found, output:

```json
[]
```

22. Do **not** enclose the JSON string in code fences and do **not** output any explanation text. Your final answer for this skill must be **only** the JSON array.