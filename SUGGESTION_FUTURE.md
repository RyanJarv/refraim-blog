# Combined Suggestions for the “Future” Section

Scope
- Target: `blog.md` conclusion/future and `FUTURE.md` roadmap.
- Basis: Common themes across SUGGESTION_FUTURE.codex.md, .claude.md, .gemini.md.
- Anchor to results: Default to `packed 5.5–7k`, precision‑first `packed_fixed 25–30k`, pragmatic `project`, context quality > size.

Structural Updates
- Use a three‑part conclusion: What We Learned; Why This Matters; What’s Next.
- Add subsections: Key Surprises; Key Takeaways (callout); Limitations; Call to Action.
- Keep the current title or split into “Conclusion” + “Future Directions”.

Near‑Term (v0.6)
- Module‑scoped chunker: group by package/dir to bridge `project` and `file`.
  - Hypothesis: better recall on cross‑file vulns than `file` with ≤1.1× runtime.
  - Metrics: F1, recall; runtime; tokens; skipped‑lines/100k.
- Overlap guardrail for long files: thresholded fallback to `fixed_token` with deterministic overlap K.
  - Hypothesis: boosts recall on long functions with small precision cost.
  - Metrics: cohort recall delta; precision impact; skipped‑lines/100k.
- Reproducibility hardening: seed/cache control; tokenizer versioning; skipped‑lines fix.
  - Acceptance: ±5% F1 variance across `--times 3` reruns.

Medium‑Term (v0.7)
- AST‑aware splitting (tree‑sitter) with graceful fallback to `syntactic`.
  - Hypothesis: precision uplift at equal token budgets on C‑like/TS/Java.
- Adaptive chunking policy: choose strategy/size from repo/file signals (filesize histograms, language mix, coupling).
  - Hypothesis: ≥3 F1 improvement vs `packed 5.5–7k` at equal token/time budget.
- Dependency/call‑graph hints: lightweight import/call edges to co‑locate related files for `packed`.
  - Goal: recall uplift on cross‑file cohorts (Juice Shop/Verademo).

Longer‑Term (v0.8+)
- Agentic two‑phase retrieval: fast coarse pass → expand high‑uncertainty hotspots within hard token/time budgets.
  - Goal: higher recall at fixed budget vs static `packed`.
- Chunker⇄Triager feedback loop: use triager outcomes to adapt chunk sizes/strategies on subsequent passes.
  - Goal: reduce FPs at equal recall; fewer re‑scans per confirmed finding.
- Incremental/diff‑aware scanning: focus on changed files with neighborhood expansion.

Evaluation Plan
- Datasets: keep current four; add synthetic “long‑function” cohort and one high‑coupling OSS repo. Optionally OWASP Benchmark/Juliet subset for function‑level signals.
- Metrics: F1, precision, recall, runtime, tokens, skipped‑lines/100k; include budgeted scores (max tokens, max wall‑time).
- Ablations:
  - Adaptive policy on/off vs best static; feature importance.
  - AST on/off at equal overlap; per‑language slice.
  - Overlap K∈{32,64,128}; threshold N sweep for fallback.
- Acceptance examples:
  - Module: +2–3 F1 on cross‑file cohort; ≤1.1× runtime.
  - Overlap: +5 recall points on long‑function cohort; ≤2 precision points loss.
  - Adaptive: ≥3 F1 uplift vs `packed 5.5–7k` at equal budgets.
  - Triager loop: ≥10% FP reduction at equal recall; ≤1.15× runtime.

Integration Details
- CLI (refraim):
  - `--strategy module | syntactic_ast | adaptive`
  - `--adaptive-policy default|rules:path.json`
  - `--budget-max-tokens`, `--budget-max-seconds`
  - `--overlap-guard on|off --overlap-tokens K --syntactic-threshold N`
- Config: mirror flags; language map for AST; budget guards; cache controls.
- Hooks (fraim): strategy registry; splitter interface with thresholded fallback; repo analysis (filesize histograms, import graph); cache keyed by file hash.
- Outputs: CSV/JSON audit log of file→strategy/size and policy decisions.

Key Surprises (to include in Conclusion)
- Project strategy competed with large `packed` while often faster.
- Syntactic vs fixed overlap tracked closely; context quality beat perfect boundaries.
- Large contexts suffered sharper recall loss than expected (context dilution).
- Strong dataset effects suggest automated codebase profiling for strategy selection.

Limitations (to include in Conclusion)
- Triage disabled in benchmarks; precision reflects within‑vulnerable‑file noise, not full‑repo FP rate.
- Single model family; results may vary across models/sizes.
- Limited datasets; generated set excluded from decisions; timings are environment‑relative.

Call to Action
- Replicate on other models/datasets; contribute strategies (module, AST, adaptive); validate with triage enabled.
- Share challenging repos; collaborate on benchmark cohorts (long‑function, cross‑file).

Suggested Snippets
- “Next: a budget‑aware adaptive policy that selects strategy and size from repository signals. We will evaluate head‑to‑head with the best static default at equal token/time budgets, with acceptance defined as a ≥3‑point F1 improvement averaged across datasets.”
- “We will introduce a module‑scoped strategy and a deterministic overlap guardrail for long functions. Success means improved recall on cross‑file and long‑function cohorts with bounded precision costs and ≤10% runtime overhead versus file‑level baselines.”
- “We will pilot a chunker⇄triager feedback loop. Success means fewer false positives at equal recall and fewer re‑scans per confirmed finding.”

References
- blog.md:234
- FUTURE.md:1

