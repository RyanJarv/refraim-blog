# Suggestions Not Included in the Combined Future Plan

These were proposed across SUGGESTION_FUTURE.{claude,codex,gemini}.md but are excluded to keep the Future plan focused on chunking strategy evolution closely tied to current blog results. Consider them optional or longer‑horizon.

Model Strategy and Fine‑Tuning
- Multi‑model escalation pipelines (fast model → stronger model) as a core roadmap item.
- Fine‑tuning a vulnerability model to reduce required context.
Reason: Valuable but outside the chunking‑first scope of current results; revisit after chunking/adaptive work lands.

Rich Vulnerability‑Specific Context Graphs
- Building tailored data‑flow/taint‑style “context graphs” per vulnerability class beyond lightweight dependency/call hints.
Reason: Likely high effort; start with import/call‑graph hints and agentic retrieval before deeper flow analysis.

Section Renaming Requirement
- Hard rename to separate “Conclusion” and “Future Directions”.
Reason: Optional; current plan works with added subsections under existing title. Adopt only if the section grows substantially.

Visual Decision Tree in Future Section
- Embedding a full strategy decision graphic at the end of the conclusion/future section.
Reason: Useful, but belongs in Results/Quick Start; the Future section should emphasize roadmap and research directions.

Community Logistics Details
- Extensive “how to join” logistics (Slack/GitHub callouts) beyond a brief Call to Action.
Reason: Keep the Future section concise; place detailed participation info in README or docs.

Alternate Heuristics Not Prioritized Now
- Vulnerability‑type‑specific chunk sizing policies (bespoke per CWE class).
- Complexity‑only chunk sizing without broader adaptive policy.
Reason: Subsumed by the broader adaptive policy work; can be explored as features within it.

Notes
- These items can be re‑introduced later if scope expands beyond chunking or if initial adaptive/AST/overlap/module work underperforms.

