# Suggestions to Strengthen the “Future” Section

Scope
- Target sections: `blog.md` conclusion/future and `FUTURE.md`.
- Goal: make future work concrete, testable, and actionable; align with Overview.md expectations and incorporate cross‑doc suggestions.

Gaps Observed
- Future work too high‑level (only “adaptive chunking” mention) without hypotheses or acceptance bars.
- Conclusion structure mixes findings and future; lacks “surprises,” “limitations,” and broader implications.
- No triager/scan feedback loop; limited view on multi‑model or fine‑tuning directions.
- Minimal roadmap and integration details (flags, config, registries) for new strategies.

Structural Updates (Conclusion/Future)
- Adopt a clear 3‑part conclusion structure: What We Learned; Why This Matters; What’s Next.
- Add subsections: Key Surprises; Key Takeaways (callout); Limitations; Call to Action.
- Title options: keep “Conclusion: The Future of Context in AI Security” but add the above subsections; or split into “Conclusion” and “Future Directions”.

Recommendations
- Add a 1‑screen “Future at a Glance” box: What (deliverables), Why (user impact), How (evaluation + integration).
- Tie each future item to a hypothesis, metric, and acceptance criteria; include risks.
- Show minimal adaptive architecture (pipeline bullets); reference budget guards and audit outputs.

Key Surprises (to include in Conclusion)
- Project strategy competed with large packed while often faster—more viable than expected.
- Syntactic vs fixed overlap tracked closely; context quality outweighed perfect boundaries.
- Precision/recall trade‑off steeper than expected at very large contexts (context dilution).
- Strong dataset effects suggest codebase profiling could drive strategy selection.

Limitations (to include in Conclusion)
- Benchmarks disable triage; precision reflects within‑vulnerable‑file noise only.
- Single model family; results may vary across LLMs and sizes.
- Datasets limited; generated set excluded from decisions; environment‑dependent timings.

Concrete Deliverables (prioritized)
- Near term (v0.6)
  - Module‑scoped chunker: package/dir grouping between `project` and `file` for better locality than `project` with less noise.  
    • Hypothesis: improves F1 vs `file` on cross‑file vulns with <10% runtime overhead vs `file`.  
    • Impl: add `module` strategy and grouping heuristics (folder/package manifests).  
    • Metric: F1, recall; runtime; tokens per run.
  - Overlap enforcement for large files: deterministic `fixed_token` overlap guardrail when syntactic splits exceed thresholds.  
    • Hypothesis: raises recall on long functions; small precision cost.  
    • Impl: thresholded fallback: if `syntactic` split >N tokens, enforce `fixed_token` overlap K.  
    • Metric: delta recall on “long‑function” cohort; skipped‑lines/100k.
  - Reproducibility hardening: cache seeds, normalize tokenization version, fix skipped‑lines edge case.  
    • Acceptance: re‑running `refraim benchmark run --times 3` yields ±5% variance on F1 across datasets.
- Medium term (v0.7)
  - AST‑aware splitting (tree‑sitter) for function/class boundaries with fallback to `syntactic`.  
    • Hypothesis: +precision on Java/TS/C‑like repos; maintains recall with overlap.  
    • Impl: optional `--strategy syntactic_ast` with language map; graceful fallback per file.  
    • Metric: precision uplift vs `syntactic` at equal token budget.
  - Adaptive chunking policy: choose strategy/size by repo/file features (filesize histogram, language mix, coupling score).  
    • Hypothesis: beats best static default by ≥3 F1 points on average.  
    • Impl: offline policy trained/rule‑based; runtime budget guard (max tokens, max time).  
    • Metric: F1, recall at fixed budget; wall time; cost (token‑estimated).
  - Dependency/call‑graph hints: lightweight static import graph to co‑locate related files in `packed`.  
    • Metric: recall uplift on cross‑file vulns in Juice Shop/Verademo.
- Longer term (v0.8+)
  - Agentic retrieval loop: iterative search‑expand on suspected hotspots with capped budget.  
    • Metric: recall at fixed token/time budget vs static `packed`.  
  - Embedding‑guided pre‑filter: coarse semantic retrieval to pre‑select candidate files for packing.  
    • Risk: embedding quality drift; add ablations and cache.
  - Incremental/diff‑aware scanning: focus on changed files with context expansion into dependency neighborhoods.
  - Chunker⇄Triager feedback loop: use triager outcomes to adapt chunk sizes/strategies on subsequent passes.  
    • Hypothesis: reduces FPs by learning failure patterns; boosts recall by escalating uncertain findings.  
    • Metric: precision uplift at equal recall; fewer re‑scans per finding.
  - Multi‑model escalation + fine‑tuning: fast model for broad scan, escalate top‑K findings to stronger model; explore fine‑tuned vulnerability model requiring less context.  
    • Metric: improved F1 at fixed cost; reduced tokens per confirmed finding.

Evaluation Plan (make it testable)
- Datasets: keep current four; add one synthetic “long‑function” cohort and one high‑coupling OSS repo. Optionally include OWASP Benchmark or Juliet subset for function‑level signals.
- Protocol: maintain current comparator; add per‑cohort reporting (long functions, cross‑file) to show targeted gains.
- Metrics: F1, precision, recall, runtime, tokens, skipped‑lines/100k; add budgeted scores (max tokens, max wall‑time) to reflect real usage constraints.
- Ablations:  
  - Adaptive: policy on vs best static; feature importance.  
  - AST: AST on vs off at equal overlap; per‑language slice.  
  - Overlap: K in {32, 64, 128} tokens; threshold N sweep for fallback switching.
- Acceptance bars (examples):  
  - Module: +2–3 F1 vs `file` on cross‑file cohort, ≤1.1× runtime.  
  - Overlap: +5 recall points on long‑function cohort at ≤2 precision points loss.  
  - Adaptive: ≥3 F1 uplift vs `packed 5.5–7k` average at same token budget.
 - Extras for new lines:  
   - Triager loop: ≥10% FP reduction at equal recall on Verademo/Juice Shop; ≤1.15× runtime.  
   - Multi‑model: ≥2 F1 improvement or ≥20% token reduction per true positive.

Integration Details (so it ships)
- CLI surface (refraim):  
  - `--strategy module | syntactic_ast | adaptive`  
  - `--adaptive-policy default|rules:path.json`  
  - `--budget-max-tokens`, `--budget-max-seconds`  
  - `--overlap-guard on|off --overlap-tokens K --syntactic-threshold N`
- Config schema (YAML/TOML): mirror flags; include language map for AST; budget guard.
- Hook points (fraim):  
  - Strategy registry: add `module`, `syntactic_ast`, `adaptive` entries.  
  - Splitters: expose common interface; allow thresholded fallback to `fixed_token`.  
  - Repo analysis: filesize histograms, import graph (simple parser per language).  
  - Cache: token counts, AST trees, import graphs keyed by file hash.
- Outputs: write policy decisions alongside chunks for auditability (CSV/JSON with file→strategy/size).
 - Optional: triager feedback channel (JSONL of outcomes → policy learner); escalation mapping for multi‑model pipelines.

User Impact Framing (tell readers why it matters)
- Security: reduce false negatives in cross‑file flows without drowning in noise.  
- Performance: meet fixed time/cost budgets with better recall than static defaults.  
- Operability: reproducible benchmarks, auditable decisions, predictable overlaps on long files.
 - Evolution: closed‑loop chunker⇄triager and multi‑model strategies bring pragmatic gains without brittle one‑off tuning.

Risks and Mitigations
- Embedding/AST drift or parser gaps → language map with fallbacks, per‑language flags, snapshot parser versions.  
- Budget blow‑ups with adaptive policies → hard caps; early stopping with “best‑so‑far” checkpoints.  
- Evaluation leakage (overfitting to datasets) → hold‑out repos, rotating test set, cohort‑based reporting.

Documentation Updates to Plan
- In `blog.md`: expand the “Future” conclusion to include:  
  - One paragraph on adaptive policy + budgeted evaluation.  
  - One paragraph on module vs project vs file and overlap guardrails.  
  - One sentence on upcoming AST support and dependency hints with risks.  
  - A 4‑bullet “What’s Next” list mapped to versions.
- In `FUTURE.md`: evolve bullets into a roadmap with hypotheses, metrics, and acceptance bars; link to CLI flags and config examples.
 - Add Key Surprises, Limitations, and a Call to Action; optionally a quick reference decision guide linking back to Results.

Suggested Snippets (drop‑in ready)
- “The next step is a budget‑aware adaptive policy that selects chunking strategy and size based on repository signals (filesize histograms, language mix, and light dependency graphs). We will evaluate it head‑to‑head against the best static default at equal token and time budgets, with acceptance defined as a ≥3‑point F1 improvement averaged across datasets.”
- “We will introduce a module‑scoped strategy and a deterministic overlap guardrail for long functions. Success means improved recall on cross‑file and long‑function cohorts with bounded precision costs and ≤10% runtime overhead versus file‑level baselines.”
 - “We will experiment with a chunker⇄triager feedback loop and a multi‑model escalation path. Success means fewer false positives at equal recall and improved F1 or reduced cost per confirmed finding.”

References (internal)
- `blog.md:234` Future conclusion wording.  
- `FUTURE.md:1` Current bullets to expand.
 - `SUGGESTION_FUTURE.claude.md`, `SUGGESTION_FUTURE.gemini.md` for structure and research‑line prompts.
