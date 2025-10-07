# Consolidated Suggestions for blog.md

Scope
- Target: blog.md and supporting references (results, info/analysis.md).
- Goal: Improve clarity, accuracy, and actionability without changing core conclusions.

Priority: Critical
- Add TL;DR and Quick Start.
  - TL;DR: Default to `packed ~5.5–7k` for best average F1 and speed. Juice Shop best F1: `packed_fixed 12k` (0.2933). Validation Benchmarks best F1: tie between `packed 5.5k` and `packed 23k` (0.4675). Verademo best F1: `packed_fixed 13k` (0.3363).
  - Quick Start: Default `packed 5.5–7k`; Precision-first: `packed(_fixed) 13–25k` (expect recall trade-offs); Speed-first on small/medium repos: `project` if memory permits; Very large single files: prefer `fixed_token`.

- Clarify `packed` vs `packed_fixed` and `original` line-based strategies on first mention.
  - `packed`: packs whole files; splits oversized files using the syntactic splitter.
  - `packed_fixed`: same packing; splits oversized files using fixed_token.
  - `original_*`: legacy line-based splits; explain trade-offs and when it’s still useful.

- Clarify Validation Benchmarks tie wherever “best F1” appears.
  - State explicit tie at 0.4675 for `packed 5.5k` and `packed 23k`.

Priority: High
- Add Method Labels cheat sheet at the start of Results.
  - `project`, `module`, `file`, `original_*`, `syntactic_*`, `fixed_token_*`, `packed_*`, `packed_fixed_*`.

- Strengthen the opening with a concrete hook.
  - Use a short, real-feeling scenario where context affected vulnerability detection.

- Add version and environment context near first results mention.
  - "Results reflect fraim v0.5.1 (branch feature-additional-chunkers) benchmark outputs; 2 runs." 

- Add an at-a-glance summary table per dataset.
  - Columns: Dataset | Best F1 | Best Precision | Best Recall | Fastest | Link to HTML report.

- Repeat key methodology caveats briefly within Results.
  - Baseline-referenced vulnerable files only; skipped lines excluded from recall; precision reflects within-file noise; triage disabled here.

- Add 1–2 concrete vulnerability examples with minimal code.
  - Show context-sensitive detection and a false positive from insufficient context.

Priority: Medium
- Conversational, active tone; align with STYLE.md (em dashes, code block language tags, remove redundant cross-references).

- Expand triage impact briefly.
  - Typical direction of change for recall/precision when triage is enabled in practice.

- Future work: make adaptive chunking more concrete.
  - Mention AST-aware splitters, learned routing, overlap enforcement, triage policies.

Suggested insertion points
- TL;DR and Quick Start: After title and intro.
- Labels cheat sheet: Beginning of Results and Analysis.
- Validation tie reminder: Dataset Behavior section for Validation Benchmarks.
- Version + paths + warning callout: Reproducibility/commands section.
- Packed vs Packed_Fixed + Module: Exploring Chunking Strategies.
- Concrete examples: End of Methodology or start of Results.

Sync requirements
- Verify all numbers match `results/results.md` and `info/analysis.md` (juice-shop best F1 now `packed_fixed 12k` at 0.2933; Validation tie at 0.4675).
- Normalize naming: prefer “Validation Benchmarks” after one initial path reference.

