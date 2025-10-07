# Benchmark Analysis — Final Conclusions

Summary
- The best default across datasets is packed chunking around 5.5–7k tokens. In the latest run, 5.5k yielded the highest average F1, with this mid‑size band continuing to balance speed and accuracy. Very large packed sizes tend to increase measured precision but reduce recall and are currently not recommended due to a skipped‑lines handling issue.
- For precision‑first workflows where the repository fits, the project strategy is a strong choice and avoids skipped‑lines; it also compares closely with large packed sizes on average and is sometimes faster. For larger repositories, defer large packed precision picks until the skipped‑lines fix lands and results are re‑validated.
- Syntactic (language‑aware) chunking tracks fixed‑token chunking closely and does not consistently outperform it. In practice, fixed_token is slightly more predictable because its overlap behavior is deterministic, whereas the language splitters used for syntactic chunking do not reliably enforce the requested overlap.
- Generated datasets validate the pipeline (near‑perfect recall) but are not decision drivers; conclusions rely on real‑repo datasets.

Headline Metrics (current results)
- Average across datasets:
  - Best F1: packed 5.5k (0.3348)
  - Best Precision: packed_fixed 25k (0.5495) — observed but not recommended until skipped‑lines fix
  - Best Recall: packed_fixed 1k (0.3913)
  - Fastest: packed 6.5k (~93.06s)
- Validation Benchmarks:
  - Best F1: packed 5.5k (ties packed 23k) at 0.4675; Best Precision: project (0.6923); Best Recall: syntactic 3k (0.5385); Fastest: packed 5.5k (~55.44s)
- Juice Shop:
  - Best F1: packed_fixed 12k (0.2933); Best Precision: packed_fixed 25k (0.5); Fastest: packed_fixed 20k (~99.50s)
- Verademo:
  - Best F1: packed_fixed 13k (0.3363); Best Precision: packed 30k (0.6471); Best Recall: packed_fixed 4k (0.4151); Fastest: packed_fixed 15k (~77.94s)

Interpretation
- Context sufficiency vs. dilution: mid‑sized packed (≈5.5–7k tokens) balances signal and noise; very large contexts tend to reduce recall despite improved precision.
- Dataset traits matter: longer spans in validation‑benchmarks benefit from mid‑large windows; heavy same‑file clustering (verademo) rewards methods that preserve intra‑file context; juice‑shop’s breadth makes size tuning more sensitive.
- Practical guidance: start with packed ~5.5–7k tokens; for precision‑first picks today, prefer project on small/medium repos (no skipped‑lines). We do not recommend very large packed sizes for precision until the skipped‑lines fix lands and results are re‑validated. Between fixed_token and syntactic, prefer fixed_token when you need predictable overlap behavior; expect parity otherwise.
 - Non‑structured cues drive hypotheses: at the scanning (pre‑triage) stage, LLMs appear to lean heavily on non‑structured information not tightly bound to AST boundaries—function/variable names, comments/docstrings, documentation and identifiers conveying intent, plus general “code‑smell” signals (complexity, awkward patterns, inconsistent naming, missing comments). This mirrors human review, which starts from threat‑model and intent before deep control‑/data‑flow. Because these cues often span or ignore strict syntax units, preserving whole files and deterministic overlap tends to matter more than perfect syntactic splits, helping explain why syntactic did not outperform fixed‑token in our runs.

Scope & Limits
- Benchmarks scan only baseline‑referenced vulnerable files; triage is disabled to highlight chunking differences.
- Skipped lines represent unscanned spans and are excluded from recall while reported separately. We observed a handling issue at very large contexts; avoid recommending those sizes for precision until the fix lands and results are re‑run.

Units
- All chunkers use token budgets except original, which uses lines.

Implementation notes
- File and Project default to using ~70% of the model’s max context to reduce near‑limit failures (FileChunker splits only when a single file exceeds the budget; Project/MaxContext packs whole files on a fixed‑token base and only splits when needed).
- We test both `packed` (syntactic split base when a single file exceeds the budget) and `packed_fixed` (fixed_token split base). Current results favor `packed ~5.5–7k` on average and on validation‑benchmarks, with `packed_fixed ~13k` edging out on verademo and `packed_fixed 1k` showing stronger small‑window recall.
