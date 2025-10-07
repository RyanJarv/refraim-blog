**Datasets That Shape The Benchmark Results**

- The comparison engine matches on `properties.type`, same-file location overlap, then semantic similarity. Dataset traits that increase ambiguity (many results of the same type in the same file) tend to lower precision/recall unless chunks retain strong local context.

**repos/juice-shop**

- Scale and mix: 75 results across 32 files (2.34 results/file avg). Top types: Sensitive Data Exposure (17), Broken Access Control (11), XSS (10), SQL Injection (9), Broken Authentication (8).
- Congestion hotspots (file | type → count):
  - `server.ts | Broken Access Control` → 4
  - `data/static/securityQuestions.yml | Broken Authentication` → 4
  - `routes/login.ts | SQL Injection` → 4
- Region size: median span 0 lines, p90 ≈ 2 lines (mostly single-line anchors).
- Why it matters: Moderate clustering per file with varied types. Chunkers that preserve immediate lines around anchors should do well; cross-file generalization helps due to breadth of categories.

**repos/validation-benchmarks**

- Scale and mix: 30 results across 20 files (1.50 results/file avg). Top types: Sensitive Data Exposure (9), Broken Access Control (4), Command Injection (3), IDOR (3), XSS (2).
- Congestion hotspots:
  - `benchmarks/XBEN-024-24/app/app.py | Sensitive Data Exposure` → 2 (others mostly 1s across files/types)
- Region size: median span ≈ 4 lines, p90 ≈ 9 lines (larger spans than other suites).
- Why it matters: Sparse per-file clustering reduces same-file ambiguity, but longer regions mean chunkers must capture wider context for recall; narrow chunks risk misses.

**repos/verademo**

- Scale and mix: 65 results across 19 files (3.42 results/file avg). Top types: XSS (32), SQL Injection (8), Sensitive Data Exposure (6), Path Traversal (5), Broken Authentication (4).
- Congestion hotspots:
  - `app/src/main/webapp/WEB-INF/views/profile.jsp | XSS` → 10
  - `app/src/main/java/com/veracode/verademo/controller/UserController.java | Security Misconfiguration` → 9
  - `app/src/main/java/com/veracode/verademo/controller/ResetController.java | Broken Authentication` → 8
- Region size: median span 0, p90 0 (highly localized single-line anchors).
- Why it matters: Heavy same-file, same-type clustering (especially XSS) increases candidate matches. Chunkers need enough template/controller context to disambiguate locations; otherwise expect false matches and missed true positives.

**Synthetic generated suites (data/generated/*)**

- Shape: 12 single-vulnerability projects (1 result in 1 file each). Types include XSS, SQL Injection, Command Injection, Path Traversal, IDOR, Sensitive Data Exposure, Broken Authentication.
- Region size: typically 0–6 lines (short spans).
- Why it matters: No ambiguity. Metrics primarily validate that a chunker can capture the single local defect; recall drops mostly indicate over-fragmentation or misaligned anchors.

**Cross‑dataset takeaways for interpreting results**

- Ambiguity driver: verademo >> juice-shop > validation-benchmarks. Expect chunkers that keep more intra-file context to score relatively better on verademo.
- Context window need: validation-benchmarks has larger spans; chunkers with bigger or adaptive windows gain recall there without harming precision (skipped lines are minimal in reported runs).
- Breadth vs. depth: juice-shop rewards generalizable chunking due to balanced type distribution; verademo rewards depth within a file due to clustering; generated sets are basic sanity checks for local anchoring.
