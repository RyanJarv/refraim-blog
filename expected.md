**Project Relation**
- Refraim benchmarks and validates Fraim’s scan output by comparing SARIF results to curated baselines using semantic matching. The blog’s code path `blog/code/fraim` references this project’s code and experiments; the CLI here runs Fraim (`fraim run code`) and evaluates against `data/**/expected.sarif`.

**What Was Reviewed**
- Baselines in: `data/repos/juice-shop/expected.sarif`, `data/repos/validation-benchmarks/expected.sarif`, `data/repos/verademo/expected.sarif`, and all `data/generated/*/expected.sarif`.
- Structural conformance to `data/expected.schema.json`, vulnerability type enums, locations/regions, duplicates, path layout, and metadata that could influence Refraim’s matcher and benchmark metrics.

**How Matching Works (Context)**
- Refraim matches results by `properties.type` and overlapping file paths, then uses semantic similarity on message/explanation. Each expected result can be matched at most once. Extra expected duplicates increase false negatives; invalid types prevent matches entirely. Multi-location expected results increase file overlap but still count as one expected finding.

**repos/juice-shop**
- Scale and mix: 75 results across 32 files. Type counts: Sensitive Data Exposure 17, Broken Access Control 11, XSS 10, SQL Injection 9, Broken Authentication 8, others smaller.
- Levels: 38 `error`, 29 `warning`, 8 `note`.
- Regions: median span 0 lines; ≈p90 ≈ 2 lines (highly localized anchors).
- Paths: URIs are relative to `code/` (e.g., `server.ts`, `routes/login.ts`, `frontend/...`).
- Duplicates: 8 duplicate result records share identical `(type, file, startLine, endLine)`, e.g., `server.ts` Broken Access Control:497; `routes/login.ts` Sensitive Data Exposure:31 (3x); SQL Injection:34 (3x). These will inflate false negatives because only one actual can match one expected.
- Schema/type issues: 2 results use `properties.type: "XEE"` instead of `"XXE"` (not in the enum). These cannot match any actual `XXE` results and will always be counted as false negatives, reducing recall.
- Confidence/exploitable: average confidence ≈ 6.96 (1–10 style); `exploitable` True:15 / False:60.
- Notes that affect benchmarks:
  - Duplicate entries and the two `XEE` typos artificially lower recall and increase false negatives for XXE/BAC/SQL/SDE hotspots.
  - Small spans make file+line matching easier; semantic text still drives final match.

**repos/validation-benchmarks**
- Scale and mix: 30 results across 20 files. Type counts: Sensitive Data Exposure 9, Broken Access Control 4, Command Injection 3, IDOR 3, XSS 2, SSRF 2, XXE 1, SQL Injection 1, Path Traversal 1, Broken Authentication 2, Security Misconfiguration 2.
- Levels: all 30 are `error`.
- Regions: median span ≈ 4 lines; ≈p90 ≈ 11 lines (larger than other suites).
- Paths: URIs are relative to `code/` (e.g., `benchmarks/XBEN-024-24/app/app.py`).
- Duplicates: none detected.
- Schema/type: all types valid.
- Confidence/exploitable: confidence is set to 90 for all results (different scale from 1–10 used elsewhere); `exploitable` True:30.
- Notes that affect benchmarks:
  - Longer spans require chunkers to preserve broader context to avoid misses; narrow chunks can hurt recall here.
  - The confidence scale differs substantially (90 vs 1–10 elsewhere). Refraim’s comparator doesn’t use confidence, but downstream tooling that inspects confidence should not mix scales.

**repos/verademo**
- Scale and mix: 65 results across 16 files. Type counts: XSS 32, SQL Injection 8, Sensitive Data Exposure 6, Path Traversal 5, Broken Authentication 4, Command Injection 2, Insecure Deserialization 2, Security Misconfiguration 2, Insufficient Logging 4.
- Levels: 48 `warning`, 9 `error`, 8 `note`.
- Regions: median span 0; p90 0 (single-line anchors dominate).
- Paths: URIs are relative to `code/app` (see `data/repos/verademo/meta.json`) and appear as `app/src/...` in the SARIF.
- Duplicates: 2 duplicate result records share identical `(type, file, startLine, endLine)`.
- Multi-location results: several results reference many locations (up to 17/11 per result). Example: one Security Misconfiguration result references 9 lines in `UserController.java`. Blog “hotspot” counts reflect location counts, not result counts.
- Schema/type: all types valid.
- Confidence/exploitable: average confidence ≈ 6; `exploitable` False for all 65.
- Notes that affect benchmarks:
  - Multi-location expected results compress many related lines into one expected. If Fraim emits multiple granular results, only one can match the single expected, and the rest become false positives; if it emits a single clustered result, matching behaves as intended.
  - The small, highly clustered spans in JSP/controllers increase same-file, same-type ambiguity; semantic text is key.

**data/generated/**
- 12 single-vulnerability projects. Each `expected.sarif` has exactly 1 result in 1 file with short spans (0–6 lines). Types include XSS, SQL Injection, Command Injection, Path Traversal, IDOR, Broken Authentication, Sensitive Data Exposure.
- No duplicates; all types valid; URIs are relative to `code/` in each project.
- Notes that affect benchmarks:
  - These are ideal for sanity checks; misses here usually indicate chunking over-fragmentation or path mapping mistakes.

**Cross‑Dataset Differences That Impact Results**
- Ambiguity and clustering: verademo >> juice-shop > validation-benchmarks. Expect chunkers that retain intra-file context to score relatively better on verademo.
- Context window need: validation-benchmarks’ longer spans benefit from larger/adaptive windows; small fixed chunks risk recall loss.
- Baseline structure: verademo uses multi-location results; juice-shop uses one location per result. Matching is per-result; multi-location entries still count as one expected.
- Severity/metadata: validation-benchmarks uses all `error`; others mix `warning`/`note`. Refraim’s comparator ignores level/confidence, but reporting/UX may differ.

**Problems Found (Actionable)**
- `data/repos/juice-shop/expected.sarif`: two `properties.type` values are `"XEE"` (invalid). Should be `"XXE"` per `data/expected.schema.json`. This causes guaranteed false negatives for XXE.
- Duplicate expected results:
  - Juice Shop: 8 duplicate entry groups (identical type+file+line). Deduplicated in-place; results reduced 75 → 64.
  - Verademo: 2 duplicate entry groups (identical type with identical location sets). Deduplicated in-place; results reduced 65 → 63.
- Confidence scale inconsistency: `data/repos/validation-benchmarks/expected.sarif` uses confidence `90` for all results; others use ≈1–10. Harmless for Refraim matching, but misleading for any downstream processes that assume a uniform scale.

**Suggested Fixes**
- Corrected `XEE` → `XXE` in `data/repos/juice-shop/gen_expected.py` and in the baseline.
- Remove exact-duplicate results in Juice Shop and Verademo baselines.
- Normalize confidence scale or clearly document scale differences if confidence is surfaced.

**References**
- Dataset overview: `blog/datasets.md`
- Baselines: `data/repos/juice-shop/expected.sarif`, `data/repos/validation-benchmarks/expected.sarif`, `data/repos/verademo/expected.sarif`, `data/generated/*/expected.sarif`
- Schema: `data/expected.schema.json`

**Update Log**
- Deduplicated duplicate expected findings in Juice Shop and Verademo baselines to prevent inflated false negatives during benchmarking.
- Added final de-dup passes to `data/repos/juice-shop/gen_expected.py` and `data/repos/verademo/gen_expected.py` with preservation of existing messages/explanations/levels for matching keys.
- Fixed `XEE` → `XXE` mapping in the Juice Shop generator and baseline.
