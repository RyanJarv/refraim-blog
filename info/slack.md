# Chunking Blog — Slack Conversation Digest (LLM‑Ready)

Purpose: Summarize the historical Slack conversations into an actionable, validated brief so the blog post reflects original intent, covers the right experiments, and avoids stale or misleading details.

## Original Intent
- Write a blog post on “chunking context for LLM code analysis.”
- Cover “chunking at all” (no chunking vs. chunking) and which chunking strategies yield the best results.
- Use or extend refraim to benchmark chunkers across datasets and models.

## Agreed Approach
- Outline → draft through results → implement/enable chunkers in fraim → run benchmarks with refraim → analyze → conclude → iterate with team.
- Investigate the impact of chunk size on detection: does performance degrade as chunk size increases? Plot the chunk size/detection curve.
- Consider semantic/AST‑aware chunking (tree‑sitter suggestion) as a future or comparative angle if feasible.

## Chunking Strategies To Cover (validated in code)
- project: Packs whole files up to a token budget (MaxContextChunker). Built on `PackingFixedTokenChunker` (fixed‑token base) and defaults to 70% of the model’s max context.
  - Path: `../../fraim/fraim/inputs/chunkers/max_context.py`
- file: One file per chunk if it fits; very large files are split using a fixed‑token splitter sized to 70% of the model’s max context to avoid API errors.
  - Path: `../../fraim/fraim/inputs/chunkers/file.py`
- original: Line‑count based splitter with light boundary heuristics.
  - Path: `../../fraim/fraim/inputs/chunkers/original.py`
- syntactic: Language‑aware splitting using LangChain’s language splitters.
  - Path: `../../fraim/fraim/inputs/chunkers/syntactic.py`
- fixed_token: Token‑based splitter using tiktoken.
  - Path: `../../fraim/fraim/inputs/chunkers/fixed.py`
- packed: Bin‑packs chunks by token budget (does not split files across packs unless a single file exceeds the budget).
  - Path: `../../fraim/fraim/inputs/chunkers/packed_fixed.py` (PackingSyntacticChunker)
- packed_fixed: Bin‑packs fixed_token chunks by tokens. Added after fixed_token showed stronger, more predictable performance.
  - Path: `../../fraim/fraim/inputs/chunkers/packed_fixed.py` (PackingFixedTokenChunker)

Names and defaults reflected in the CLI defaults:
- Path: `../src/refraim_cli/benchmark.py`

## Measurement Units (validated)
- fixed_token, packed, packed_fixed, project: token‑based sizing (CodeChunk/CodeChunks length uses token counts via tiktoken).
- original: line‑based sizing.
- file: token‑based cutoff when splitting (70% of model context); otherwise single‑file chunks.

Implication: When plotting or comparing “chunk_size” across methods, be explicit about units; only `original` uses lines.

## Known Issue (clarified)
- Earlier notes suggested `packed` used character counts. This is obsolete. `CodeChunk`/`CodeChunks` implement `__len__` using token counts, so `packed` and `packed_fixed` both honor token budgets. Keep unit labels clear; do not imply character limits.

## Open Questions / To Validate
- AST/semantic chunking: Tree‑sitter‑based function/class chunking was suggested. Not implemented in current fraim code. If included, scope as future work or a note comparing with “syntactic” (language‑aware) splitting.
- Chunk‑size curve: Confirm if performance drops at very large chunks (context dilution) and at very small chunks (missed cross‑file signals). Empirically show F1/precision/recall vs. size.
- Runtime: Benchmarks are slow; document typical runtime ranges and any mitigation (subset datasets, fewer repeats) used to iterate.

## Practical Plan (for the blog workflow)
- Datasets: Use existing refraim datasets (generated/*, repos/juice-shop, repos/verademo, etc.).
- Tests: Include project, file, original, syntactic, fixed_token (several sizes/overlaps), packed (token sizes), and packed_fixed.
- Metrics: precision, recall, F1, processing time, skipped lines.
- Output: HTML per dataset + average.html + markdown summary (`blog/results/`).
- Reporting: Clearly label units; group bars by chunking method; note exclusions in averages (e.g., generated/* if applicable in code).

## Risks / Constraints
- Mixed units (tokens vs. lines) across chunkers can mislead if unlabeled (`original` uses lines).
- AST chunking is aspirational; do not imply it’s implemented unless done.
- Slower run times can limit grid search granularity; document selected sizes and rationale.

## Action Items
- Ensure units are explicit in figures and tables (tokens vs. lines).
- If using a “best size,” state the search space and selection criteria (metric priority and tie‑breakers).
- If including AST chunking, call it out as experimental or “future work” unless implemented.
- Document any prior confusion about units; confirm tokens for all but `original`.

---

Appendix: Abridged raw Slack excerpts (for traceability)

- Request: Blog post on chunking context for LLMs; cover “chunking at all” and which chunking yields best results; leverage refraim for benchmarking.
- Plan (Ryan): Outline → draft; propose testing process and chunkers; integrate chunkers; produce results/analysis; iterate with team.
- Ideas: Explore other projects’ chunking; solicit chunker ideas.
- Suggestion (David): Semantic/AST chunking (tree‑sitter). Study how chunk size affects detection; produce size/detection curve. “Plan of attack sounds good.”
- Update (Sep 20): Early confusion around packed measurement was clarified; all chunkers use token budgets (except `original`, which is line‑based). Tests were re‑run with consistent labeling.

End of digest.
