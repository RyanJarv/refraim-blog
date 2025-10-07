# Results Section — Simplification Suggestions (at‑a‑glance focus)

Scope
- Target: `blog.md` → “Results and Analysis” and adjacent tables/graphs.
- Goal: Make trade‑offs and best‑choice recommendations instantly scannable; reduce repetition; keep numbers authoritative and linked to `results/results.md`.

Priority Changes
- Lead with a single "Key Finding" line, then a 4‑line TL;DR card.
  - Key Finding: Mid‑sized packed (5.5–7k) balances accuracy (best avg F1 0.3348) and speed; larger contexts raise precision but lower recall.
  - Default: Use `packed 5.5–7k` for best average F1 and speed.
  - Precision‑first: Use `packed_fixed 25–30k` (accept lower recall).
  - Recall‑first: Use `packed_fixed 1–4k` (accept lower precision).
  - Speed on small/medium repos: Consider `project` when memory permits.

- Consolidate insights into a single “Quick Picks” matrix.
  - Columns: Goal | Recommendation | Why | Caveat
  - Rows: Best F1 (avg), Highest precision, Highest recall, Fastest, Small repo speed/accuracy, Very large single file.

- Replace multi‑section narrative with one “Key Patterns” box.
  - Up‑size → precision increases, recall decreases (context dilution).  
  - Down‑size → recall increases, precision decreases (more noise).  
  - `project` ≈ large `packed` accuracy on some sets, often faster.  
  - `packed` is best overall default; `packed_fixed` yields steadier overlap and top precision at large budgets.

- Present a one‑screen “Decision Flow.”
  - Need speed on small/med repo? → `project`.
  - Need balanced default? → `packed 5.5–7k`.
  - Need max precision? → `packed_fixed 25–30k`.
  - Need max recall (triage later)? → `packed_fixed 1–4k`.
  - Large single file exceeds budget? → split with `fixed_token`.

- Reframe around reader goals (jobs‑to‑be‑done).
  - "If your priority is speed..." → `project`
  - "If you need highest precision..." → `packed_fixed 25–30k`
  - "If you need highest recall..." → `packed_fixed 1–4k`
  - "If you want the best balance..." → `packed 5.5–7k`

- Collapse and relocate caveats.
  - Move “Un‑Scanned Lines Correction”, “Precision scope”, and matching rules into a compact Footnotes/Caveats list right after TL;DR.
  - Keep one line each; link “Details” anchor further down if needed.

- Trim and merge overlapping sections.
  - Merge “Method‑Specific Insights”, “Cross‑Method Insights”, and “What We Expected vs. Found” into “Key Patterns” + “Quick Picks”. Keep only one tight list of bullets.
  - Keep dataset‑specific one‑liners; remove duplicate phrasing.

- Prefer three primary subsections for skimmability.
  - The Winners (just the winners + one‑line recommendation)
  - Performance Trade‑offs by Strategy (focused 3–4 strategy comparison)
  - Why These Results Matter (connect back to the research questions)

Table/Graphs
- Simplify the At‑a‑Glance table.
  - Columns: Dataset | Best F1 | Best Precision | Best Recall | Fastest | Recommendation.
  - Move HTML report links to a short “Reports” list immediately below the table (or footnotes).
  - Bold metric winners; add subtle badges: “Default”, “Low recall trade‑off”, “Lower precision trade‑off”.
  - Consistent naming: “Validation Benchmarks”, “Juice Shop”, “Verademo”, “Generated”.

- Add a single visual “Performance Dashboard”.
  - Option A (scatter): F1 vs Processing Time; each point is a configuration; highlight `packed 5.5k` and `project`.
  - Option B (grouped bars): For each dataset, bars for Best F1, Best Precision, Best Recall (from the table).
  - Keep iframes below; add image fallback thumbnails + links to the HTML.

- Add quick legend + how to read.
  - “Right/up is better; larger bubble = tokens; color = method; hollow markers = recall‑first picks.”

Clarity/Consistency
- Define method labels once, near the top of Results.
  - `project` (whole repo), `module` (package group), `file` (per‑file), `original_*` (line‑based), `syntactic_*` (language‑aware), `fixed_token_*` (deterministic overlap), `packed` (packs whole files; splits big ones syntactic), `packed_fixed` (same, but fixed_token for splits).

- Normalize tokens and decimals.
  - Use the en dash range “5.5–7k”; keep two decimals for metrics; round times to 2 decimals.

- Put v0.5.1 context at the start of Results.
  - “Numbers reflect fraim v0.5.1; mirrored from `results/results.md`.”

- Remove cross‑refs like “see Results and Analysis”.
  - Replace with local anchors or inline summaries.

Recommended Rewrite Outline
1) Key Finding + TL;DR (4 bullets) + Caveats (3 one‑liners)
2) Quick Picks matrix (Goal → Rec → Why → Caveat) + Decision Flow
3) Visual Performance Dashboard + legend; links to detailed reports
4) At‑a‑Glance table (dataset bests) + short “Reports” list
5) Strategy Comparison (packed, project, fixed_token, syntactic) with “When to use” lines
6) Key Patterns (5–6 bullets total) and “Why this matters”
7) Dataset Notes (1 line each)
8) Repro/Appendix (cost warning; commands; environment; move timing/skip details here)

Proposed Text Snippets
- TL;DR block
  > TL;DR — Default to `packed 5.5–7k` for the best average F1 and strong speed. For precision‑first reviews, bump to `packed_fixed 25–30k` (expect lower recall). For recall‑first hunts (triage later), use `packed_fixed 1–4k`. On small/medium repos where memory allows, `project` offers a pragmatic speed/accuracy trade‑off.

- Caveats block (tight)
  - Scan scope: only baseline‑referenced vulnerable files; precision reflects within‑file noise.
  - Skipped lines: spans excluded from recall; reported separately; fix is planned.
  - Matching: same vuln type + overlapping file; best semantic match on text/snippet.

- Key Patterns (copy/paste)
  - Larger contexts raise precision but reduce recall; smaller contexts invert that trade‑off.
  - `packed 5.5–7k` is the best default on average; `packed_fixed 13k` edges verademo.
  - `packed_fixed 25–30k` yields the highest precision; expect recall to drop.
  - `project` compares with large `packed` on some datasets and often runs faster.
  - For single files exceeding the budget, prefer `fixed_token` for predictable overlap.

- Quick Picks (example rows)
  - Goal: Best average F1 → Rec: `packed 5.5–7k` → Why: balance of precision/recall → Caveat: mid‑size only.
  - Goal: Highest precision → Rec: `packed_fixed 25–30k` → Why: fewer false positives → Caveat: lower recall.
  - Goal: Highest recall → Rec: `packed_fixed 1–4k` → Why: more hits → Caveat: noisier, triage recommended.
  - Goal: Fastest overall → Rec: `packed 6.5k` → Why: lowest average time → Caveat: dataset dependent.
  - Goal: Small/medium repo speed → Rec: `project` → Why: whole‑repo context without packing → Caveat: memory/cost.
  - Goal: Very large single file → Rec: `fixed_token` base → Why: deterministic overlap → Caveat: structure not preserved.

- Dataset notes (1‑liners)
  - Validation Benchmarks: best F1 ties `packed 5.5k` and `packed 23k` (0.4675); precision rises with size; recall falls.
  - Juice Shop: cross‑file vulns; small chunks miss cues; very large contexts mix tests and prod (dilutes recall).
  - Verademo: mirrors Juice Shop trends; `packed_fixed 13k` edges best F1.
  - Generated: sanity check; near‑perfect by construction (don’t over‑weight).

- Strategy Comparison (copy/paste scaffold)
  - Packed (Recommended default)
    - Sweet spot: 5.5–7k (F1 0.3348)
    - Trade‑off: 25–30k boosts precision (~0.55) with lower recall
    - When to use: start here for most codebases
  - Project (Speed‑optimized on small/medium repos)
    - Performance: close to large packed; often faster (e.g., ~192s avg)
    - When to use: speed matters and memory permits
  - Fixed‑token base (Fallback for large single files)
    - Strength: deterministic overlap; predictable splits
    - When to use: very large single files or when syntactic splitting is unreliable
  - Syntactic (Language‑aware)
    - Note: tracks fixed‑size; can underperform slightly; overlap not always enforced
    - When to use: prefer boundary preservation when splitters are accurate

- Decision Flow (text version)
  - Is speed critical? → Yes → Choose `project`; No → Continue
  - Is there a very large single file? → Yes → Use `fixed_token` base; No → Continue
  - Do you need max precision? → Yes → `packed_fixed 25–30k`; No → Continue
  - Default: `packed 5.5–7k`

- “What we found in 30 seconds” (optional opening)
  - Best overall: `packed 5.5–7k` (F1 ≈ 0.33)
  - Fastest accurate: `project` on small/medium repos (~192s avg)
  - Highest precision: `packed_fixed 25k` (~0.55)
  - Avoid extremes: very small (<3k) or very large (>30k) unless goal‑driven

Implementation Hints
- Keep the long narrative in an Appendix if desired; the main Results should fit on a single screen before scrolling.
- Use callouts/badges sparingly for “Default pick”, “Precision‑first”, “Recall‑first”.
- Add an image fallback for each iframe so static renderers see something useful.
- Ensure all numbers match `results/results.md`; prefer referencing that file programmatically when possible.
 - Bold winners; number strategies (1–6) for reference; add horizontal rules between major blocks.
 - Move secondary metrics (e.g., skipped lines per 100k, processing times tables) to Appendix; summarize in one line in Results.
 - Create a short “Reports” list with links to `results/*.html` instead of a table column.
 - Keep overlapping descriptions minimal (syntactic vs fixed_token); defer details to footnotes/appendix.
