# Consolidated Suggestions for Blog Results Section

This document merges the common, high‑value recommendations from:
- `SUGGESTION_RESULTS.codex.md`
- `SUGGESTION_RESULTS.gemini.md`
- `SUGGESTION_RESULTS.claude.md`

It prioritizes clarity and actionability for the Results section in `blog.md`, aligned with the actual benchmark winners and trade‑offs reported in `results/results.md`.

## What To Add (Consensus + Highest Impact)

- Lead with a concise TL;DR that answers “what should I do?”
- Provide a Quick Picks matrix that maps user goals to exact configurations.
- Include a simple decision flow to make a choice in seconds.
- Define method labels once near the top for consistent terminology.
- Compress methodology caveats into a short “Important Context” block.
- Add a single “hero” visual (F1 vs Processing Time) with a one‑line legend.
- Keep an at‑a‑glance winners table and add small “recommendation” badges.
- End with one‑line dataset notes to ground expectations by repo type.
- Note version/scope context and add a short cost warning.

## Copy‑Ready Snippets

1) TL;DR block

> TL;DR
> - Default: `packed 5.5–7k` → best average F1 (≈0.3348) and strong speed
> - Precision‑first: `packed_fixed 25–30k` → highest precision (≈0.55), lower recall
> - Recall‑first: `packed_fixed 1–4k` → highest recall (≈0.39), noisier
> - Small/medium repo speed: `project` → competitive accuracy, ~192s avg

2) Method Labels (place once near TL;DR)

Method labels guide
- `project` = whole‑repo input
- `file` = per‑file chunks
- `original_*` = line‑based splitter
- `syntactic_*` = language‑aware splitting
- `fixed_token_*` = deterministic token overlap
- `packed` = packs whole files, splits oversized syntactic
- `packed_fixed` = packs whole files, splits oversized with fixed_token

3) Important Context (tight caveats)

- Numbers reflect fraim v0.5.1; triage disabled for these benchmarks.
- Scope: scans baseline‑referenced vulnerable files; precision reflects within‑file noise.
- Matching: same vuln type + overlapping file; best semantic match on text/snippet.

4) Quick Picks matrix (Goal → Rec → Why → Trade‑off)

| Goal | Recommendation | Why | Trade‑off |
| --- | --- | --- | --- |
| Best overall balance | `packed 5.5–7k` | Best avg F1 and solid speed | Mid‑size only |
| Highest precision | `packed_fixed 25–30k` | Fewer false positives | Lower recall |
| Highest recall | `packed_fixed 1–4k` | More hits found | Noisier; triage recommended |
| Fastest overall | `packed 6.5k` | Lowest avg time (~93s) | Dataset dependent |
| Small/medium repo speed | `project` | Whole‑repo context | Memory/cost limits |
| Very large single files | `fixed_token` base | Predictable overlap | Structure not preserved |

5) Decision Flow (one‑screen)

Need speed on a small/medium repo? → Yes → Use `project` (~192s avg)
  ↓ No
Are there very large single files? → Yes → Use `fixed_token` base
  ↓ No
Need max precision? → Yes → `packed_fixed 25–30k`
  ↓ No
Need max recall? → Yes → `packed_fixed 1–4k`
  ↓ No
Default → `packed 5.5–7k`

6) Key Patterns (replace overlapping narrative)

- Larger contexts tend to raise precision while reducing recall; smaller contexts invert that trade‑off.
- `packed 5.5–7k` is the best default on average; large `packed/packed_fixed` boosts precision at recall’s expense.
- `project` can compete with large `packed` on accuracy and often runs faster on small/medium repos.
- `syntactic` generally tracks `fixed_token`; overlap isn’t always enforced by splitters.
- Prefer `fixed_token` when dealing with very large single files for predictable overlap.

7) At‑a‑Glance winners (keep compact, add badges)

- Average across datasets
  - Best F1: `packed 5.5k` (0.3348) [DEFAULT]
  - Best Precision: `packed_fixed 25k` (0.5495) [PRECISION‑FIRST]
  - Best Recall: `packed_fixed 1k` (0.3913) [RECALL‑FIRST]
  - Fastest: `packed 6.5k` (~93.06s)

- Validation Benchmarks
  - Best F1: tie `packed 5.5k` / `packed 23k` (0.4675)
  - Best Precision: `project` (0.6923)
  - Best Recall: `syntactic 3k` (0.5385)

- repos/juice‑shop
  - Best F1: `packed_fixed 12k` (0.2933)
  - Best Precision: `packed_fixed 25k` (0.5)
  - Fastest: `packed_fixed 20k` (~99.50s)

- repos/verademo
  - Best F1: `packed_fixed 13k` (0.3363)
  - Best Precision: `packed 30k` (0.6471)
  - Best Recall: `packed_fixed 4k` (0.4151)
  - Fastest: `packed_fixed 15k` (~77.94s)

8) Hero chart + legend (optional but recommended)

- Chart: Scatter of F1 (y) vs Processing Time (x); bubble size = tokens; color = strategy.
- Annotate key picks: `packed 5.5k`, `packed 6.5k`, `project`, `packed_fixed 25k`.
- Legend one‑liner: “Right/up is better; larger bubble = more tokens.”

9) Version and cost callouts

> Results reflect fraim v0.5.1 with triage disabled.
> Cost: larger contexts (25k+) increase API cost; consider pricing tier.

## What To Trim or Move (Alignment Across Inputs)

- Merge overlapping “Method‑Specific/Cross‑Method/Expected vs. Found” into compact Key Patterns.
- Keep long narrative, timing details, and secondary metrics in an Appendix.
- Avoid repeating the same winner multiple times; mention once in TL;DR and reference the table.
- Keep numbers authoritative by mirroring `results/results.md`.

