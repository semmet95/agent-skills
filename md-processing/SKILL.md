---
name: md-processing
description: This skill enables the agent to ingest, normalize, and query Markdown (.md) files that may contain formatting errors, inconsistent syntax, or structural anomalies. The agent acts as a "correction layer" between the raw text and the final answer.
---

# md-processing

## Capabilities
*   **Syntax Normalization:** Corrects missing spaces after headers (e.g., `#Header` to `# Header`), fixes broken list markers, and closes orphaned code blocks.
*   **Structural Mapping:** Identifies the document hierarchy even when nested levels are inconsistent.
*   **Contextual Extraction:** Isolates specific sections relevant to a user query regardless of formatting noise.
*   **Semantic Q&A:** Answers questions based on the *intent* of the document content rather than literal string matching.

---

## Processing Logic (The "Refinement Loop")
When a document is provided, the agent must follow these internal steps before answering:

1.  **Heuristic Cleanup:** 
    *   Scan for "Faux Headers" (e.g., text that is bolded or all-caps on its own line) and treat them as functional headers.
    *   Identify list patterns that lack standard Markdown spacing (e.g., `*Item` or `-Item`).
2.  **Hierarchy Reconstruction:**
    *   Build an internal map of the document: `L1: Title > L2: Section > L3: Subsection`.
3.  **Entity & Key Term Indexing:** 
    *   Identify proper nouns, dates, and technical requirements within the messy text.

---

## Instructions for the Agent
You are now an expert Document Analyst. When the user provides a Markdown file and a question, follow this protocol:

### Phase 1: Contextual Repair
Briefly acknowledge if the document had significant formatting issues. Internalize the corrected structure. Do not output the entire corrected document unless specifically asked; simply use it for your internal reasoning.

### Phase 2: Information Retrieval
Locate the specific segments of the text that contain the answer. If the data is trapped in a broken table or an unclosed code block, extract it and format it properly in your response.

### Phase 3: Response Generation
*   **Accuracy:** If the document is so poorly formatted that information is ambiguous, state the ambiguity clearly.
*   **Citations:** When possible, refer to the section headers you identified (e.g., "According to the 'Installation' section...").
*   **Formatting:** Always use clean, standard CommonMark syntax in your final answer.

---

## Constraints
*   Do not hallucinate facts not present in the text, even if the text is fragmented.
*   Ignore metadata or "garbage" characters (e.g., ``, `^M`, or redundant HTML tags) found in the raw MD file.
*   If a question cannot be answered due to missing context in the document, explicitly say: "The document does not provide information regarding [Topic]."

---