# Benchmark Analysis — Final Conclusions

Summary
- The best default across datasets is packed chunking around 5.5–7k tokens. In the latest run, 5.5k yielded the highest average F1, with this mid‑size band continuing to balance speed and accuracy. Larger packed sizes increase precision but reduce recall.
- The project strategy compares closely with large packed sizes on average and is sometimes faster, making it a pragmatic option when speed matters and memory permits.
- Syntactic (language‑aware) chunking tracks fixed‑token chunking closely and does not consistently outperform it. In practice, fixed_token is slightly more predictable because its overlap behavior is deterministic, whereas the language splitters used for syntactic chunking do not reliably enforce the requested overlap.
- Generated datasets validate the pipeline (near‑perfect recall) but are not decision drivers; conclusions rely on real‑repo datasets.

Headline Metrics (current results)
- Average across datasets:
  - Best F1: packed 5.5k (0.3348)
  - Best Precision: packed_fixed 25k (0.5495)
  - Best Recall: packed_fixed 1k (0.3875)
  - Fastest: packed 6.5k (~93.06s)
- Validation Benchmarks:
  - Best F1: packed 5.5k (0.4675); Best Precision: project (0.6923); Best Recall: syntactic 3k (0.5385); Fastest: packed 5.5k (~55.44s)
- Juice Shop:
  - Best F1: fixed_token 20k (0.2687); Best Precision: packed_fixed 25k (0.5); Fastest: packed_fixed 20k (~99.50s)
- Verademo:
  - Best F1: packed_fixed 13k (0.3363); Best Precision: packed 30k (0.6471); Best Recall: packed_fixed 4k (0.4038); Fastest: packed_fixed 15k (~77.94s)

Interpretation
- Context sufficiency vs. dilution: mid‑sized packed (≈5.5–7k tokens) balances signal and noise; very large contexts tend to reduce recall despite improved precision.
- Dataset traits matter: longer spans in validation‑benchmarks benefit from mid‑large windows; heavy same‑file clustering (verademo) rewards methods that preserve intra‑file context; juice‑shop’s breadth makes size tuning more sensitive.
- Practical guidance: start with packed ~5.5–7k tokens; consider larger packed (15–30k, especially packed_fixed) when precision is paramount and recall degradation is acceptable; consider project on small to medium repos where speed is critical. Between fixed_token and syntactic, prefer fixed_token when you need predictable overlap behavior; expect parity otherwise.

Scope & Limits
- Benchmarks scan only baseline‑referenced vulnerable files; triage is disabled to highlight chunking differences.
- Skipped lines represent unscanned spans and are excluded from recall while reported separately; they are generally low in these runs.

Units
- All chunkers use token budgets except original, which uses lines.

Implementation notes
- File and Project default to using ~70% of the model’s max context to reduce near‑limit failures (FileChunker splits only when a single file exceeds the budget; Project/MaxContext packs whole files on a fixed‑token base and only splits when needed).
- We test both `packed` (syntactic split base when a single file exceeds the budget) and `packed_fixed` (fixed_token split base). Current results favor `packed ~5.5–7k` on average and on validation‑benchmarks, with `packed_fixed ~13k` edging out on verademo and `packed_fixed 1k` showing stronger small‑window recall.
