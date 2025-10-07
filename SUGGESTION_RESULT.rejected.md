# Rejected or De‑prioritized Suggestions

These items appeared in one or more `SUGGESTION_RESULTS.*.md` files but were not included in `SUGGESTION_RESULT.md`. Reasons are given to clarify the trade‑offs.

## Presentation/Structure

- Separate single‑line “Key Finding” before TL;DR (Codex)
  - Rationale: TL;DR already carries the headline guidance; avoids duplication.

- Enforce a strict “Three‑Act Structure” (Gemini)
  - Rationale: We consolidated around TL;DR → Quick Picks → Flow → Patterns → Table; explicit three‑act framing is optional.

- Implementation priority checklist (Claude)
  - Rationale: Useful for planning, but not essential to the consolidated guidance for the blog.

## Tables/Formatting Details

- Redefine At‑a‑Glance table columns and move HTML links to a separate “Reports” list (Codex)
  - Rationale: Table column changes and a reports list are editorial niceties, not core to the results narrative.

- Bold winners, normalize tokens/decimals, enforce en‑dash ranges and 2‑decimal rounding (Codex)
  - Rationale: Style guidance only; we prioritized structural clarity and correctness over formatting specifics.

## Visuals

- Color legend mapping (e.g., Blue=packed, Green=fixed_token, Orange=syntactic) (Claude)
  - Rationale: Color choices depend on the actual chart implementation; kept a generic legend.

- Image fallback thumbnails for iframes (Codex)
  - Rationale: Nice to have; excluded to keep the core guidance concise.

## Extras/Callouts

- Audience‑specific callout boxes (e.g., “For Security Engineers”) (Gemini)
  - Rationale: Not a common request across sources; can be added later if needed.

- Anchors and cross‑reference cleanup details (Codex)
  - Rationale: Editorial hygiene; omitted from the core consolidated recommendations.

## Implementation Hints (Deferred)

- Programmatic number sourcing from `results/results.md` (Codex)
  - Rationale: Agreed in spirit (we call for mirroring numbers), but the implementation detail is left to maintainers.

- Numbered strategy lists and inter‑block separators/HRs (Codex)
  - Rationale: Formatting preference; not critical to content.

