# Results Explanations — Updates on Packed vs. Packed_Fixed and Recent Chunker Changes

Update (current results):
- We now test both `packed` and `packed_fixed`. Both use a bin‑packing strategy; they differ only in how they split a single oversized file before packing:
  - `packed` uses the syntactic splitter (language‑aware boundaries).
  - `packed_fixed` uses the fixed‑token splitter (deterministic token windows).
- File chunker behavior was aligned with project defaults: it includes a single file per call when possible, but will split very large files using the fixed‑token splitter sized to ~70% of the model’s max context to avoid near‑limit errors.
- Project (MaxContextChunker) uses a fixed‑token base (via `PackingFixedTokenChunker`) and also defaults to ~70% of the model’s context.

Packed vs. Packed_Fixed — observed patterns (from results/results.md):
- Average across datasets:
  - Best F1: `packed 5.5k` = 0.3348 (vs. best `packed_fixed` 5k = 0.3134)
  - Best Precision: `packed_fixed 25k` = 0.5495
  - Best Recall: `packed_fixed 1k` = 0.3875 (vs. `packed 1k` = 0.3452)
- Validation‑benchmarks:
  - Best F1: `packed 5.5k` = 0.4675 (vs. `packed_fixed 7k` = 0.4138; `packed_fixed 5k` = 0.4045)
  - Best Recall: `syntactic 3k` = 0.5385
  - Best Precision: `project` = 0.6923
- Verademo:
  - Best F1: `packed_fixed 13k` = 0.3363
  - Best Precision: `packed 30k` = 0.6471
  - Best Recall: `packed_fixed 4k` = 0.4038
- Juice Shop:
  - Best F1 overall: `fixed_token 20k` = 0.2687 (packed vs. packed_fixed are close in F1 around mid sizes; `packed_fixed 25k` yields highest precision 0.5 with reduced recall.)

Takeaways:
- When most files fit under the token budget, both packed variants behave similarly (full‑file packs). Differences appear when large files must be split before packing.
- `packed` (syntactic base) tends to win on average and on validation‑benchmarks at mid sizes (~5.5–7k), and provides strong large‑budget precision.
- `packed_fixed` (fixed‑token base) tends to provide steadier recall at small windows and slightly better performance on `verademo` at ~13k, consistent with token‑predictable overlaps in same‑file‑clustered code.
- Default guidance remains: start with `packed ~5.5–7k` for balanced F1 and speed; consider `packed_fixed ~13k` for verademo‑like same‑file clustering or prioritize small‑window recall (`packed_fixed 1k`).

Note: The following historical explanation section remains for context from earlier runs and focuses on why certain mid‑size single‑file strategies scored well. Some concrete figures below refer to a prior run and may differ from the current tables.

**Key Context**
- The benchmark scans only baseline‑referenced vulnerable files, not entire repositories (`src/refraim/benchmark.py`:131). This reduces cross‑file noise and favors “whole‑file” style chunks.
- The comparator matches when vuln `type` is equal and there is at least one overlapping file among locations; then it picks the best semantic match (`src/refraim/sarif/compare.py`:49, `src/refraim/sarif/compare.py`:133).
- False negatives inside unscanned regions are excluded from recall (tracked via parsing notifications and the `scanned` flag). Skipped lines are reported separately (`src/refraim/sarif/compare.py`:146, `src/refraim/lib/_results.py`:191).

**Validation‑Benchmarks** (earlier run overview retained for historical context)
- `original 1.3k`: F1 0.5455, Precision 0.4468, Recall 0.7, Skipped 0
- `syntactic 6k`: F1 0.4533, Precision 0.3778, Recall 0.5667, Skipped 0
- `fixed_token 6k`: F1 0.4722, Precision 0.4048, Recall 0.5667, Skipped 0

Why these are high:
- Right‑size fit: Medium windows (6k tokens or ~1.3k lines) usually encompass an entire single‑purpose XBEN case, avoiding fragmentation while not mixing unrelated cases.
- No skip artifact: Skipped lines = 0 for these rows, so higher recall is not from excluding false negatives.

**Juice Shop** (earlier run overview retained)
- `file`: F1 0.1779, Precision 0.1894, Recall 0.1678, Skipped 0.002
- `original 1.3k`: F1 0.3529, Precision 0.375, Recall 0.3333, Skipped 0
- `syntactic 6k`: F1 0.4000, Precision 0.3846, Recall 0.4167, Skipped 0
- `fixed_token 6k`: F1 0.3966, Precision 0.3966, Recall 0.3966, Skipped 0.003

Why the same pattern appears:
- The benchmark still scans only vulnerable files. Many of those files fit entirely within these windows (e.g., `server.ts` is 729 lines), so the model sees “complete file” context without unrelated repo noise. This mirrors the advantage seen on validation‑benchmarks.

**Why not “file” strategy?**
- Although “file” also processes one chunk per vulnerable file, the prompt structure and budget discipline of `original`/`syntactic`/`fixed_token` lead to clearer, more matchable outputs. In practice, that improves both recall and precision relative to a raw whole‑file feed.
- Comparator behavior does not give “file” any advantage; it only needs one overlapping file to match, so better per‑file messages from right‑sized chunkers translate directly into more true positives.

**Takeaways (earlier run)**
- Elevated scores align with dataset structure (single‑file or file‑centric cases) and the benchmark’s “vulnerable files only” scope.
- Mid‑sized, single‑chunk‑per‑file strategies (original 1.3k, syntactic 6k, fixed_token 6k) can be strong on some runs; current results favor `packed ~5.5–7k` overall with `packed_fixed` better on verademo at ~13k.

**Recent Changes Audit**
- Added configs to default sweep: `original 1.3k`, `syntactic 6k`, `fixed_token 6k` (src/refraim_cli/benchmark.py:166, src/refraim_cli/benchmark.py:168, src/refraim_cli/benchmark.py:178). This surfaced their (legitimate) strong performance; it did not alter how they work.
- Baseline cleanup applies to all methods equally:
  - data/repos/juice-shop/expected.sarif (64 results) and data/repos/verademo/expected.sarif (63 results) were deduplicated; Juice Shop also fixed `XEE`→`XXE`. These reduce duplicate‑driven FNs and improve fairness for every method.
- Comparator/metrics stability benefits all methods:
  - Reuse expected keys for TP aggregation (src/refraim/sarif/compare.py:123–131), best‑match selection by type+any file overlap (src/refraim/sarif/compare.py:182–208), and correct skipped‑lines accounting (src/refraim/lib/_results.py:191).
- No chunker logic was changed here; chunkers run via the external `fraim` CLI. Nothing in this repo targets only `original 1.3k`, `syntactic 6k`, or `fixed_token 6k` beyond adding them to the test list.

Conclusion (updated): No recent code changes uniquely favor only the mid‑size single‑file strategies. The current bests mainly reflect dataset structure, packing behavior at mid sizes (~5.5–7k), and the split‑base differences (syntactic vs fixed‑token) when handling long files.
