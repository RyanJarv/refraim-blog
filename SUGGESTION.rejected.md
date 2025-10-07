# Rejected or Deferred Suggestions

The following items were considered but not included in the consolidated SUGGESTION.md either because they are duplicative, lower priority, or too specific for the current revision.

- Removed/deprioritized meta-list (from one suggestion set).
  - Reason: Our consolidated list already reflects priority; repeating removals adds noise.

- Normalize dataset naming note ("prefer consistent Validation Benchmarks").
  - Reason: Folded into “Sync requirements” in the main suggestions (naming normalization). Keeping here would duplicate guidance.

- Clarify the “~70% of max context” heuristic specifics.
  - Reason: Model- and setup-dependent; better handled in code/docs than the blog post. If included, treat as a brief aside rather than a core improvement.

- Optional embedded charts beyond links.
  - Reason: Nice-to-have; links to HTML reports plus a summary table deliver most value with less maintenance burden.

- Dataset characteristics table (size, counts) as a separate artifact.
  - Reason: Potentially useful, but secondary to the per-dataset summary table. Can be added later if needed.

- Expanded layered explanation per strategy beyond a short “When to use”.
  - Reason: The blog’s flow benefits from brevity; deep layering can be added in docs.

- Extensive cost modeling with precise $ estimates.
  - Reason: Costs vary widely by provider and tier. We include a light cost lens instead.

- Add missing Module strategy coverage.
  - Define module-based chunking (between whole project and per-file) with a short "When to use" note.

- Add warning callout before benchmark/repro commands about API cost.
  - Note `--times 3` multiplies calls; suggest rate limits and a rough cost estimate template.

- Clarify graph output paths in the reproducibility section.
  - From repo root use `--output-dir blog/results`; from inside `blog/` use `--output-dir results`.

- Reproducibility details and variance.
  - Hardware specs, seeds/sampling note if applicable, number of runs (3), and note that metrics are single-point with variance.

- Brief cost lens alongside runtime.
  - Rough tokens per strategy/size and a coarse cost estimate.

