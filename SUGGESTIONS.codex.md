# Suggested Improvements for blog.md (Consolidated v0.5.1)

Scope
- Audience: engineers and security practitioners evaluating chunking strategies.
- Goal: maximize scannability and actionability while keeping numbers aligned with `results/results.md` and analysis context.

Critical (must do)
- Add TL;DR and Quick Start near the top.
  - TL;DR: default to `packed 5.5–7k` (best average F1, strong speed); precision‑first → `packed_fixed 25–30k`; recall‑first → `packed_fixed 1–4k`; small/medium repo speed → `project`.
  - Quick Start: 4–6 bullets that map goals to configurations with trade‑offs.
- Add “When to use” guidance for every strategy.
  - One sentence essence + one line “When to use” per strategy; practical and consistent.
- Clarify Validation Benchmarks tie anywhere “best F1” appears.
  - “Best F1: `packed 5.5k` (ties `packed 23k`) at 0.4675.”
- Define method labels once at the start of Results.
  - `project`, `module`, `file`, `original_*`, `syntactic_*`, `fixed_token_*`, `packed`, `packed_fixed`.
- Clarify key terms on first mention.
  - `packed` vs `packed_fixed`; explain `original` (line‑based) succinctly; clarify “fraim” vs `refraim`.
- Add version and cost context close to the first results.
  - “Numbers reflect fraim v0.5.1; mirrored from `results/results.md`.” Add a warning callout before running benchmarks (API cost, rate limits).
- Cover “Module” strategy or note deferral.
  - If not benchmarked, explain why and where it fits; otherwise add to strategy list with a “When to use”.

High (clarity and usability)
- Make Results skimmable with two structures:
  - Quick Picks matrix: Goal | Recommendation | Why | Caveat.
  - One‑screen Decision Flow mapping scenarios to strategies (speed, precision, recall, large file).
- Keep and tighten the At‑a‑Glance table.
  - Dataset | Best F1 | Best Precision | Best Recall | Fastest | Report link.
  - Add a 1‑line “how to read the graphs”; link dataset names to `results/*.html`.
- Add 1–2 concrete vulnerability examples to show context effects.
- Repeat critical caveats inside Results.
  - Scope (baseline‑referenced files), skipped‑lines handling, matching rules, triage disabled.
- Strengthen opening hook and use active, conversational tone per STYLE.md.
- Clarify graph output paths in Reproducibility.
  - From repo root use `--output-dir blog/results`; from inside `blog/` use `--output-dir results`.
- Provide image fallbacks for iframes and a short legend for visuals.
  - Legend: “Right/up better; bubble=size; color=method; hollow=recall‑first.”

Medium (polish and depth)
- Add a compact dataset characteristics table.
  - File count (approx), typical file size, cross‑file coupling, notable traits.
- Add a brief cost lens next to runtime.
  - Rough tokens per strategy/size and approximate $ at current pricing.
- Include hardware/runs/variance notes for reproducibility.
  - Machine specs, `--times 3`, variance caveat; note seeds if relevant.
- Clarify precision scope earlier and once more in Results.
- Summarize triage impact briefly (directional, not deep dive).
- Add a “Threats to Validity” box (baseline‑only, dataset bias, model choice).
- Normalize tokens and decimals; typography consistency (em‑dashes, ranges like “5.5–7k”).
- Make future work more concrete (adaptive chunking, overlap enforcement, AST‑aware splitting).
- Explain the “~70% max context” heuristic (model‑dependent, may adapt).
- Normalize dataset naming (consistently “Validation Benchmarks”, “Juice Shop”, “Verademo”, “Generated”).

Suggested insertion points
- TL;DR + Quick Start: after intro.
- Labels guide: first paragraph of Results.
- Quick Picks + Decision Flow: top of Results; At‑a‑Glance table follows.
- Dataset notes (1‑liners): below the table; examples nearby.
- Version/cost warnings + path clarifications: Reproducibility section.

Proposed snippets (copy/paste)
- TL;DR
  > TL;DR — Default to `packed 5.5–7k` for the best average F1 and strong speed. Precision‑first: `packed_fixed 25–30k` (expect lower recall). Recall‑first: `packed_fixed 1–4k` (noisier; triage recommended). For small/medium repos, `project` offers a pragmatic speed/accuracy trade‑off when memory permits.

- Quick Start bullets
  - Best average F1 → `packed 5.5–7k` → Balanced P/R → Mid‑size only.
  - Highest precision → `packed_fixed 25–30k` → Fewer FPs → Lower recall.
  - Highest recall → `packed_fixed 1–4k` → More hits → Triage noise.
  - Fastest overall → `packed 6.5k` → Lowest avg time → Dataset‑dependent.
  - Small/medium repo speed → `project` → Whole‑repo context → Memory/cost.
  - Very large single file → `fixed_token` base → Deterministic overlap → Weaker structure.

- Method labels guide
  - `project` (whole repo), `module` (package group), `file` (per‑file), `original_*` (line‑based), `syntactic_*` (language‑aware), `fixed_token_*` (deterministic overlap), `packed` (packs whole files; splits oversized with syntactic), `packed_fixed` (splits oversized with fixed_token).

- Visuals legend
  - “Right/up is better; larger bubble = tokens; color = method; hollow markers = recall‑first picks.”

- Graph paths note
  - Paths are relative to your current working directory. From the repo root use `--output-dir blog/results`. From inside `blog/` use `--output-dir results`.

- Warning callout
  > Warning: Benchmarks with `--times 3` issue multiple API calls and can incur non‑trivial cost, especially at larger chunk sizes. Check pricing and consider rate limits.

- Version context
  - Results reflect fraim v0.5.1 benchmark outputs; values mirror `results/results.md` and `results/*.html`.

Notes on scope (after reading all suggestions)
- Keep “layered explanation” targeted to strategy pages/sections only; avoid over‑layering across the whole post (consensus trade‑off for a technical audience).
- Keep triage discussion brief and focused; the primary scope remains chunking.
- Prefer concise tables and one‑screen summaries over lengthy narrative; move detail to appendix where needed.
