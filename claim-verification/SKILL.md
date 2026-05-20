---
name: claim-verification
description: >
  Given a link to a media post containing a falsifiable claim, extract the core claim, use web search tools to find up to two high-quality external documents that directly support or refute it, and output only the URLs of those documents.
tags: [news, fact-checking, falsifiability, web-search, evidence-retrieval]
---

# claim-verification


## Goal

Take a **falsifiable claim** made by a news or media outlet as input (via its URL), understand the claim from that page, then use web search tools to find up to **two external documents** (e.g., government or institutional reports, official press releases, authoritative news articles, or media posts) that **directly and conclusively** support or refute the claim.

The skill must **only output the URLs** of the selected documents, with no extra text.

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

All other necessary information should be obtained via the web search / fetch tools.

---

## Outputs

You must output **only a JSON array of strings**, where each string is a URL, e.g.:

[
  "https://example.org/document-1",
  "https://example.com/press-release-2"
]

### Constraints:

- The array may contain **0, 1, or 2 URLs**, but **never more than 2**.
- Each URL must correspond to a **real, retrieved document** that directly addresses the claim.
- Do **not** include any other text, explanation, comments, or metadata outside this JSON array.
- Do **not** include titles, verdicts, or snippets in the output – only URLs.
- Do **not** surround the json string in a code block.

If you cannot find any suitable evidence that directly proves or disproves the claim, output an **empty array**:

[]

---

## Definitions

- **Falsifiable claim**  
  A concrete assertion about reality that can, in principle, be proven true or false by evidence (e.g., “X happened on date Y”, “Organization Z reported N cases”, “Law A was passed on date B”).

- **Conclusive supporting document**  
  A source that **explicitly and directly confirms** the claim’s core factual content (e.g., official statistics matching the numbers, official press release stating the same event and details).

- **Conclusive refuting document**  
  A source that **explicitly and directly contradicts** the claim (e.g., correction or retraction from the same outlet, official data that shows the claim is false, fact-checkers clearly rating the claim false).

> You must not rely on your own internal knowledge; all evidence must come from retrieved documents.

---

## Step-by-step instructions

### 1. Retrieve and understand the claim

1. Use the web search or fetch tool to access `claim_url`.
   - If a direct “fetch URL” tool is available, prefer it for this step.
   - Otherwise, query the web search tool with the exact URL or the article’s title + outlet name to locate the canonical page.
2. From the page content:
   - Identify the **main falsifiable claim** the article is making.
   - Focus on the headline and the lead paragraphs, which usually contain the central factual assertion.
3. Rewrite this claim **internally** (in your own reasoning) as a single precise sentence.
   - Do not output this sentence; it is for your own understanding.
4. Check that the claim is **falsifiable**:
   - It must be concrete (who/what/when/where/how much).
   - It must be testable in principle using external evidence.
   - If it is purely opinion or too vague, you will likely not be able to find conclusive documents; be prepared to output `[]`.

#### Claim Handling Rules

- Identify the core factual assertion in the original post.
- Strip away framing, adjectives, and editorial language.
- If the claim is compound, isolate the main testable statement.
- If the claim is too vague to verify, return no result or a best-effort pair only if the claim can still be meaningfully tested.

### 2. Design targeted search queries

5. Break the claim into key components:
   - Entities (people, organizations, places).
   - Actions/events (what is claimed to have happened).
   - Timeframes and quantities (dates, counts, percentages, amounts).
6. Construct **2–4 focused web search queries** combining:
   - The core entities and action.
   - Keywords like `"official statement"`, `"press release"`, `"report"`, `"fact check"`, `"data"`, or `"statistics"`, as appropriate for the claim.
7. Your goal is to uncover **primary or highly credible secondary sources**:
   - Government or intergovernmental organizations.
   - Official company or institutional press releases.
   - Reputable, independent news outlets.
   - Established fact-checking organizations.
   - Example query patterns:
      - `"[exact statistic or event]" site:gov`
      - `"[official name]" press release [date]`
      - `"[claim subject]" fact check OR investigation`

### 3. Use web search tools to gather candidate sources

8. Use the web search tool with your queries to retrieve **candidate pages**.
9. For each promising result:
   - Use the search or fetch tool to load the page.
   - Check if the page **directly addresses the same claim**:
     - It should mention the same core event, data point, or decision.
     - It should have enough detail to confirm or contradict the claim.
10. Discard:
    - Pages that only loosely mention the topic without addressing the specific claim.
    - Forum threads, low-quality blogs, or user-generated content with unclear credibility, unless nothing better exists.

### 4. Select up to two conclusive documents

11. Among the candidate sources, identify those that **most directly and explicitly** speak to the claim.
12. Prefer sources in this order:
    1. Official/public institutional sources (government sites, intergovernmental organizations, courts, regulators, central banks, statistical agencies, official company press centers).
    2. Major reputable news organizations and established fact-checkers.
    3. High-quality specialized outlets or expert organizations.
13. Choose **up to two** sources that:
    - Clearly **confirm** the claim (conclusive supporting documents), or
    - Clearly **contradict** the claim (conclusive refuting documents), or
    - One of each, if available.
    - Prefer documents from **differing sources** (e.g., one government source and one reputable news wire) to maximize evidentiary breadth, provided both are conclusive.
14. If you find more than two strong candidates:
    - Select the **two most authoritative and directly relevant** ones.
15. If you cannot find any source that directly addresses the claim:
    - Prepare to output an empty array `[]`.

#### Evidence Selection Rules

Prefer sources in this order:

1. Official press releases, filings, government records, transcripts, datasets, or direct statements
2. Primary reporting from reputable news outlets
3. Original media posts or direct public records
4. Secondary articles only if no stronger source exists

A source is valid only if it directly addresses the claim.

Choose sources that:

- Are independent of each other
- Are directly relevant to the claim
- Are more authoritative than the original media post
- Are as conclusive as possible

Avoid:

- Duplicate reports of the same underlying source
- Circular citations
- Opinion pieces
- Weakly related sources
- Sources that only mention the topic in passing

### 5. Avoid hallucinations and fabricated sources

16. You must **never fabricate URLs, titles, or publishers**.[web:1281][web:1283][web:1286]
    - Every URL you output must correspond to a page you actually retrieved via the tools.
    - Do not guess URLs based on patterns (e.g., `/press-release-2024-01-01`); only use real, observed links.
17. If you are unsure about a page’s relevance or authenticity, **do not output it**.
    - It is better to output fewer links or `[]` than to provide hallucinated or misleading sources.

### 6. Produce the final output (URLs only)

18. Collect the URLs of the selected documents (maximum two).
19. Construct a **JSON array of strings** containing only those URLs, for example:

[
  "https://official.gov/statistics/report-1234",
  "https://news.example.com/article/5678"
]

20. If you have selected only one conclusive document, output:

[
  "https://example.com/only-document"
]

21. If you cannot find any suitable documents that directly prove or disprove the claim, output:

[]

22. Do **not** include:
    - Any explanatory text.
    - Any keys other than the URLs in the array.
    - Any markdown or code fences around the JSON.

Your final answer for this skill must be **only** the JSON array of URLs.

