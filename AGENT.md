# Blog Agent Guide

This folder supports the blog post in `blog.md`. It documents how to refresh benchmark results, where to find assets, and what to read to finish the post.

## Purpose & Scope
- Summarize benchmark results of the fraim vulnerability scanner, produced by the refraim tooling in this repo.
- Refraim drives fraim against datasets in `../data/` and compares SARIF to baselines.
- Some older notes may be outdated; prefer the sources listed below.

## Read This First
- fraim background: `./code/fraim/AGENTS.md`
- refraim background: `./code/refraim/AGENT.md`
- Blog overview (canonical outline): `Overview.md`
- Current draft: `blog.md`
- Process and internals: `info/process.md`
- Conversations and decisions: `info/slack.md` (team Slack excerpts about the blog’s scope, chunkers, and workflow)
- Datasets and traits: `datasets.md`
- Syntactic splitting probe: `info/syntactic_splitting.txt` (shows overlap behavior; useful caveats)
- Recent summary tables: `results/results.md` and the per‑dataset HTML files in `results/`
- Canonical analysis details: `info/analysis.md` (basis for “Results and Analysis” in the blog)

## Folder Layout
- `blog.md` — draft post
- `datasets.md` — dataset narratives and traits
- `info/` — supporting notes, analysis, and prior summaries
- `code/` — symlinks used for inline code references in the post
- `results/` — outputs from the graph command (`*.html`, `average.html`, and `results.md`)

## Refresh Benchmarks & Graphs
Details omitted here. Use the CLI help and repo documentation to run benchmarks and render graphs when needed.

## What To Include In The Blog
- Define each chunker succinctly: `project`, `file`, `original`, `syntactic`, `fixed_token`, `packed`.
- Cite high‑level findings consistent with current results:
  - Packed (e.g., `packed_7000_700`) improves precision with stable F1 across datasets.
  - Too much context dilutes recall; too little context misses cross‑file signals.
  - Generated sets are easy (near‑perfect recall); verademo is hardest (same‑file clustering).
- Embed/inline the latest tables from `results/results.md` and figures from `results/*.html` (export PNGs if needed).
- Call out the “skipped lines” metric and what it means (see `process.md`).

## Structure Conventions (follow the Overview)
- Source of truth for outline: `Overview.md`.
- Maintain the Overview’s structure unless the user explicitly approves a change. If a change is approved, update the Overview doc accordingly so future work stays aligned.
- Section intent:
  - “Exploring Chunking Strategies”: describe how each chunker works and state the hypothesis/expected effects (no detailed results here beyond light qualitative context). Avoid hard metrics in this section.
  - “Results and Analysis”: present quantitative findings, dataset‑level and method‑level comparisons, timings, and cross‑method insights. Anchor claims with numbers from `results/results.md`.
  - “Methodology”: keep benchmark scope, comparator mechanics, and limitations.
- Generated datasets are for sanity checks; do not use them to drive conclusions. Prefer real‑repo datasets for insights.

Analysis Source of Truth
- Use `info/analysis.md` as the canonical final conclusions of the benchmark analysis. It should contain the concise findings and recommendations that the blog’s “Results and Analysis” and “Conclusion” sections reflect.
- When the understanding of results changes (new runs or insights), update `info/analysis.md` first, then mirror those conclusions in `blog.md` to keep them in sync.

Note: If diverging from these conventions (e.g., adding a new section or materially changing flow), ask the user first and record the approved outline changes in the Overview doc.

## Using Overview.md To Drive blog.md

Overview.md is the canonical outline and scaffolding for the blog. Use it to shape the narrative and ensure consistency across iterations.

- Section "Description": Treat as the intent and required coverage for that section of the blog. Make sure the blog text fulfills these bullets; expand with current results, examples, and citations to `results/` where relevant.
- "Simplified Draft" notes: Use as seed ideas or example phrasing only. They are not binding. Rewrite for clarity and accuracy, keeping the promised content from the adjacent "Description".
- "Hypothesis": Include the original hypothesis verbatim or paraphrased clearly in blog.md. Explicitly compare expectations vs. observed results, and explain meaningful divergences. This contrast is valuable to readers.
- Important: Do not modify the Hypothesis text in `Overview.md`. It is treated as frozen source material for the draft. Keep "When to Use" as its own subsection in `Overview.md` (do not merge it into Hypothesis). Reflect expectation vs. results in the blog’s Results section instead.
- "When to Use": Present expected use cases, then update them based on actual results. If real findings contradict the original guidance, surface the change and explain why. Calling out the delta from the original expectation is encouraged.
- "Actual Results" (when present): Verify against current outputs (`results/results.md`, per‑dataset HTML) and align with `info/analysis.md`. Update any stale notes here in the blog copy; do not alter Overview.md without approval.
- Results and Analysis → "Questions": Use these to frame the analysis narrative and figures. If the "Questions" subsection is missing or outdated in Overview.md, propose additions/edits and seek explicit approval before updating the outline.
- Results and Analysis → "Test Metrics": Ensure the blog reports and interprets these metrics consistently with the current pipeline. Required metrics: Precision, Recall, F1, Processing Time, and Skipped Lines. Keep units explicit (tokens for all chunkers except `original` which uses lines). When applicable, include the scaling label for skipped lines (e.g., "Skipped Lines / 100k").
- Methodology alignment: Reflect the benchmark scope (scan only baseline‑referenced files; triage disabled), comparator semantics, and units policy. Link claims to `results/results.md` and keep high‑level conclusions synchronized with `info/analysis.md`.

Quick workflow checklist
- Read `Overview.md` to confirm section goals and hypotheses.
- Draft or revise the corresponding blog sections, using "Simplified Draft" bullets as optional prompts.
- Ground findings in `results/results.md` and mirror conclusions from `info/analysis.md`.
- Verify units and labels (tokens vs. lines; skipped lines scaling) in text and figures.

## Overview Summary (for blog work)

Purpose
- Balance context sufficiency and context dilution for LLM vulnerability scanning; measure how chunking impacts precision, recall, F1, speed, and skipped lines.

Datasets (baseline traits)
- Generated: 12 single‑vuln projects; one result per project; sanity checks only.
- repos/juice-shop: 75 results across 32 files; varied types; hardest due to real‑world scale and cross‑file relations.
- repos/validation-benchmarks: 30 results across 20 files; longer spans; unrelated test cases when concatenated.
- repos/verademo: 65 results across 19 files; heavy same‑file clustering (e.g., XSS in JSP templates).

Chunking methods (mechanics and intent)
- project: Whole‑project concatenation up to token window; preserves cross‑file relations; risk of unrelated noise.
- file: One file per chunk; no intra‑file splitting; simple baseline.
- original: Line‑based splits; sizes in lines.
- fixed_token: Token‑based splits; sizes in tokens.
- syntactic (language‑aware): LangChain language splitters snap to function/class/block boundaries; not AST‑based; requested `chunk_overlap` not reliably enforced.
- packed: File‑boundary‑aware packing to efficiently fill token budget; splits a file only if it exceeds `chunk_size` (then uses syntactic for that file).
- packed_fixed: Same packing rule; splits oversized file with fixed_token.

Testing & CLI workflow
- Benchmarks run fraim against datasets with triage disabled; scan only baseline‑referenced files.
- Metrics: precision, recall, F1, processing time, skipped lines (unscanned spans excluded from recall).
- Comparator: match on `properties.type` + at least one overlapping file, then semantic similarity over message/explanation; one expected can match at most once.

Observed patterns (latest results)
- Mid‑sized packed (≈7k tokens) yields the best average F1; larger packed sizes raise precision but reduce recall.
- Project compares closely with large packed sizes on average and can be faster; dataset‑dependent on large, clustered repos.
- Syntactic tracks fixed‑size and sometimes underperforms slightly; overlap not reliably enforced by the splitter.
- Generated suites validate the pipeline but are not decision drivers.

Future work
- Module‑scoped chunker (package‑level “project”).
- AST/embedded chunking (tree‑sitter + embeddings) for semantic retrieval.
- Adaptive/recursive strategies combining methods.

## Finishing Checklist
- Fill any incomplete sections in `blog.md` (e.g., “Project (Baseline)”).
- Verify dataset narratives against `datasets.md` counts/types.
- Ensure figures/tables match `results/*` and the CLI defaults shown in code.
- Add a short “limitations” note on scanning only vulnerable files and semantic matching.

## Direction & Non‑Goals
- Direction: Evaluate chunking strategies (project, file, original, syntactic, fixed_token, packed, packed_fixed) for LLM vulnerability scanning; produce reproducible results and keep this guide aligned with the latest outputs in `results/`.
- Non‑Goals: Implementing an AST/tree‑sitter‑based chunker, changing triage behavior, or modifying comparator semantics. Benchmarks run with triage disabled to surface chunking effects. Generated datasets are sanity checks, not decision drivers.

## Units
- All chunkers use token budgets except `original`, which uses line counts. Keep labels explicit in text and figures; do not imply character‑based sizing.

## TODO Tracking (blog/TODO.md)
- Purpose: Capture a small set of important, high‑level goals that improve or complete `blog.md` but won’t be addressed immediately.
- Keep it brief and stable. Do not track granular edits or maintain a running checklist.
- Use it for directional items (e.g., “clarify units policy in figures,” “revisit largest chunk sizes after skipped‑lines fix”).
- Update it only when major direction changes or after substantial new results; otherwise leave as‑is.
