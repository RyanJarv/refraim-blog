# Testing Process Reference

## End-to-End Flow
- Benchmarks are orchestrated through [`refraim_cli/benchmark.py`](../src/refraim_cli/benchmark.py) which calls the helper workflow in [`src/refraim/benchmark.py`](../src/refraim/benchmark.py).
- Each benchmark iteration builds a `fraim run code` command via `build_fraim_args()`, pointing at the dataset's `code/` directory, disabling triage, and limiting `--paths` to files listed in the dataset's baseline (`Dataset.vulnerable_file_paths()`).
- The scan writes a transient SARIF file into a scratch directory. `Benchmark.run()` loads this into a `Run` model (`refraim.lib._models.Run`) and pairs it with the dataset's `expected.sarif` path.
- The actual-versus-expected comparison happens lazily through `BenchmarkResult.comparison`, which loads the baseline and calls `refraim.sarif.compare.compare()` to classify findings as true positives, false positives, or false negatives.
- Metrics (`precision`, `recall`, `f1_score`, skipped lines) are computed from the resulting `RefraimComparison` object (`src/refraim/lib/_results.py`). Aggregation across runs relies on `RefraimComparison.__add__`, so repeated runs can be summed before calculating metrics.
- Results are cached on disk under `RESULTS_DIR / "benchmark"` (`refraim.lib._common.RESULTS_DIR` resolves to the OS-specific Click app dir, e.g. `~/Library/Application Support/refraim/results` on macOS). Each run lands in `<fraim version>/<dataset>/<arg slug>/<run id>/result.json` and can be reloaded without re-executing the scan.

## Dataset Inventory
`list_datasets()` enumerates `data/*/*/expected.sarif`, so the benchmark surface covers both synthetic and real-world suites. Every dataset exposes:

- `code/`: source tree to scan.
- `expected.sarif`: baseline SARIF describing ground-truth vulnerabilities.
- Optional metadata (e.g., `meta.json`, helper scripts) describing how the baseline was produced.

### Generated Suites (`data/generated`)
Auto-generated single-vulnerability projects, each validated and documented in `meta.json`. Every baseline contains exactly one `Run` with a single `Result` entry.

| Dataset | Vulnerability Type | Findings |
| --- | --- | --- |
| `go_web_server_34b3c8` | Hardcoded Secrets | 1 |
| `html_javascript_web_server_33cd67` | Cross-site Scripting (XSS) | 1 |
| `html_web_server_4a6433` | Cross-site Scripting (XSS) | 1 |
| `java_web_server_79495d` | SQL Injection | 1 |
| `javascript_web_server_130f2d` | Command Injection | 1 |
| `javascript_web_server_c8e92d` | SQL Injection | 1 |
| `php_web_server_d8b428` | SQL Injection | 1 |
| `python_web_server_2c7823` | Path Traversal | 1 |
| `python_web_server_3b37fe` | Hardcoded Credentials | 1 |
| `python_web_server_88a2f8` | Insecure Direct Object Reference (IDOR) | 1 |
| `python_web_server_be862f` | SQL Injection | 1 |
| `ruby_web_server_200a50` | SQL Injection | 1 |

Each `meta.json` records the vulnerability prompt, review status, and descriptive text used inside `expected.sarif`. A typical baseline entry (from `python_web_server_2c7823`) looks like:

```json
{
  "message": {"text": "Project 'Path Traversal - Python' successfully validated."},
  "level": "error",
  "locations": [
    {
      "physicalLocation": {
        "artifactLocation": {"uri": "app.py"},
        "region": {"startLine": 23, "endLine": 23},
        "contextRegion": {"startLine": 23, "endLine": 23}
      }
    }
  ],
  "properties": {
    "type": "Path Traversal",
    "confidence": 10,
    "exploitable": true,
    "explanation": {
      "text": "Create a Python Flask application ... allowing for sequences like '../' to access files outside the intended directory."
    }
  },
  "codeFlows": []
}
```

### Realistic Repositories (`data/repos`)
Larger multi-finding corpora sourced from real or curated projects. Baselines contain dozens of results covering diverse weakness classes.

| Dataset | Findings | Vulnerability Types |
| --- | --- | --- |
| `juice-shop` | 75 | XSS, SQL Injection, Broken Access Control, etc. (Angular/Node OWASP Juice Shop) |
| `validation-benchmarks` | 30 | 11 categories including Path Traversal, SSRF, Broken Authentication |
| `verademo` | 65 | Broken Authentication, SQL Injection, XSS, Insecure Deserialization, etc. |

Supporting files (e.g., `CLAUDE.md`, `GEMINI.md`, `gen_expected.py`) document how each baseline is maintained. For Juice Shop, `gen_expected.py` parses vulnerability markers in `code/` and emits relative URIs validated against `data/expected.schema.json`.

## Expected Results Format
- All baselines comply with the custom schema in `data/expected.schema.json`, which enforces SARIF 2.1.0 structure plus refraim-specific `Result.properties` (type, confidence, exploitable flag, explanation text).
- URIs are relative paths (no leading `file://`) so they can be joined with the dataset’s `code/` directory. `Dataset.__init__` validates that every referenced file exists.
- Each `Result` must expose at least one location with line numbers, enabling targeted scanning (via `--paths`) and aligning comparisons to actual code regions.
- Optional `codeFlows` arrays are supported but most baselines leave them empty today.

## Producing Actual Results
- Benchmarks call `fraim run code --no-triage --output=<tmp> --location=<dataset>/code --paths <relative files>`. Additional chunking flags are appended from the test matrix (e.g., `--chunk-size=7000 --chunk-overlap=700`).
- The fraim scan emits a SARIF file with a single run whose `tool.driver` metadata reflects the fraim build. Sample fixture (`tests/data/real.sarif`) includes 26 results with types such as Path Traversal.
- Invocation telemetry inside the SARIF run captures parser failures. `compare()` inspects `invocations[].toolExecutionNotifications` with descriptor id `PARSING_ERROR` to collect `unscanned_code` spans; false negatives entirely inside those spans are later marked `scanned=False` so they don’t count against recall.

## Comparison Mechanics
- `compare()` first groups expected results by a stable `finding_key` (`artifact uri + vuln type`).
- For each actual result, `Matcher.check()` enforces matching vulnerability type and at least one overlapping file before calculating semantic similarity.
- Semantic similarity combines:
  - Sentence embeddings (SentenceTransformers `all-MiniLM-L6-v2`) over result messages and explanations.
  - Optional CodeBERT embeddings over code snippets when snippets exist.
  - A small constant bonus for file-level agreement.
  The final score must meet the adaptive `similarity_threshold` (default 0.85).
- Matched pairs become `RefraimTruePositive` instances with both expected and actual payloads. Unmatched actuals produce `RefraimFalsePositive`; unmatched expected results produce `RefraimFalseNegative`.
- `RefraimComparison` exposes:
  - `precision = TP / (TP + FP)`
  - `recall = TP / (TP + adjusted FN)` where adjusted removes unscanned findings
  - `f1_score`
  - `csv()` output with columns `category,file,line,rule,message`
  - `skipped_lines` (sum of unscanned regions’ line spans)

## Benchmark Matrix and CLI Utilities
- Default benchmark test set (`DEFAULT_TESTS`) sweeps project-level, file-level, fixed-size, syntactic, fixed-token, and packed chunkers with varying sizes/overlaps.
- `refraim_cli benchmark run` parameters:
  - `--datasets`: optional subset (auto-completes from `list_datasets_names()`)
  - `--chunking-method` or arbitrary CLI overrides passed through
  - `--model`: LLM identifier (defaults to `gemini/gemini-2.5-flash`; requires `GEMINI_API_KEY`)
  - `--times`: repeat count per configuration (default 3)
  - `--output-file`: stream textual summaries (`BenchmarkResult.__str__` emits metrics summary and key)
- `refraim_cli benchmark query` reads persisted `result.json` files and prints findings filtered by dataset/test prefix, with optional JSON or verbose detail (includes `RefraimResult.detail`). Totals at the end leverage `Benchmark.stats()` to sum comparisons.
- `refraim_cli benchmark graph` converts stored stats into Plotly bar charts per dataset plus an averaged view (skipping grouped glob datasets defined in `GRAPH_GROUPS`). Artifacts land under `RESULTS_DIR/graphs/<fraim version>/`.
- `refraim_cli benchmark clear` wipes cached benchmark results.
- `refraim_cli benchmark dir` prints the resolved results directory for manual inspection.

## Managing Baselines and Fixtures
- `refraim_cli data list|info|expected|sarif` inspects datasets, dumps baseline metadata, or prints individual finding keys using `get_finding_key()`.
- `tests/data/` supplies lightweight SARIF fixtures for unit tests (e.g., `real.sarif` for integration-style comparisons, `one_result.sarif`/`no_results.sarif` for CLI smoke testing).
- `tests/test_cli.sh` exercises the CLI wrapper by running `refraim sarif compare` against the fixture pair.
- `tests/test_lib_results.py` and `tests/test_csv.py` validate `all_inside()` logic, CSV export, and result classification behavior.

## Key Takeaways for Extending Tests
- New datasets must drop an `expected.sarif` under `data/<group>/<name>/` with relative URIs and pass `Dataset` validation.
- When expanding chunking experiments, add entries to `DEFAULT_TESTS` or pass custom flags through the CLI; persisted results make reruns idempotent.
- Baseline maintenance scripts (e.g., `data/repos/juice-shop/gen_expected.py`) should ensure consistency with `expected.schema.json` and keep `properties.type` aligned with the vocabulary enforced by comparison heuristics.
- Because `RefraimComparison` weights semantic similarity heavily, descriptive `message`/`explanation` text in both baselines and scan outputs materially affects matching quality.
