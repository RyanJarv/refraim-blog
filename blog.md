# **Optimizing LLM Context for Vulnerability Scanning**

## **Introduction: The Challenge of Code Context in AI Security**

Large Language Models (LLMs) are increasingly being applied to vulnerability scanning. Their effectiveness depends heavily on how much of the surrounding code context they can see. This raises two critical questions:

1. **Does an LLM lose accuracy when fed too much code at once?** Current wisdom suggests that very large contexts dilute the model’s focus, leading to more missed detections.  
2. **How much surrounding code does an LLM need to detect a vulnerability in the first place?** Too little context and the model may lack the signals to even spot a flaw.

These questions pull in opposite directions. More context helps the model see important relationships, but too much unrelated code can overwhelm it. Our experiments benchmarked chunking strategies with these questions in mind—seeking the balance between enough context to catch vulnerabilities and not so much that the model becomes less effective.

## **Our Approach: The AI-Powered Vulnerability Scanner**

Our system, **fraim**, uses a two-stage workflow. First, it chunks repositories into smaller inputs so an LLM can scan them for vulnerabilities. Then, a **triager stage** validates those findings with full repository access, filtering out false alarms.

The scanner targets common vulnerabilities such as SQL injection, XSS, and CSRF across multiple languages (Python, JavaScript, Java, C, etc.). In these cases, context is critical. A snippet that looks unsafe in isolation may be harmless in its actual usage—or vice versa. This is why finding the right chunking method is so important.

## **The Mechanics of Code Chunking for LLMs**

Chunking means splitting a large codebase into smaller segments that fit within an LLM’s token window. It’s not only about size constraints; it’s about **feeding the model the right information**.

The key idea is to maximize useful context while minimizing irrelevant details. Done poorly, chunking can cut off the information the model needs to reason about a function. Done well, it lets the model focus on vulnerabilities in context without distraction.

## **Exploring Chunking Strategies**

Our research covers several chunking strategies and attempts to answer the following questions: 

* What if we didn't chunk at all? 
* What happens when we start combining files?
* What are the effects of semantic chunking, which aims to preserve logical code structures?
* How do different-sized chunks affect chunking strategies?


It's important to note that in all the tests below, only vulnerable files were scanned; whole‑project scans exclude ancillary documentation and files believed unrelated to the known vulnerabilities.

### **How The Tests Work**

See Methodology for dataset inventory, comparison mechanics, labels/units, and metrics definitions.

### **Project (Baseline)**

This strategy attempts to add the entire project in a single call or combines files to fill the LLM’s context window. In practice, it packs whole files up to about 70% of the model’s max context and only splits a single file that exceeds the budget using the fixed_token splitter (MaxContextChunker on a fixed‑token base). We originally predicted this would degrade performance overall for most codebases, while potentially performing well on small repositories. With the latest runs, project shows a competitive accuracy/speed trade‑off against large packed sizes; see Results and Analysis for metrics.

### **File**

The file strategy does not perform concatenation but instead attempts to fit the entire file in a single call by default; however, if the file is very large, it is split across several calls using the same fixed_token approach sized to about 70% of the model’s max context to avoid near‑limit failures. We originally hypothesized that the performance will vary depending on file size and internal code organization and that it may perform adequately for small self-contained files but fail to provide appropriate context while struggling with large files or vulnerabilities requiring cross-file context.

Compared to project, the file strategy suffered in precision but performed better in recall and, as expected, did better on the validation-benchmarks dataset which contains many unrelated sub-projects. When looking at the F1-score on other datasets, performance was relatively poor, with file generally below the stronger strategies.

### **Fixed-Size Chunking**

Fixed-size chunking splits code into calls of a set number of tokens without combining multiple files. For fixed-size, the chunk size to use becomes an important question due to the trade-off between larger contexts (potential dilution) and providing enough context to hypothesize a vulnerability.

Our hypothesis here was that testing this method across a range of chunk sizes would surface a “sweet spot” where accuracy is maximized and false positives minimized. Too small, and critical context is lost; too large, and noise increases or token limits are hit. In practice, the F1-score does not trace a smooth curve because critical context can fall in and out of chunk boundaries as sizes change, and because many files are smaller than the larger chunk sizes. This produces apparent “randomness” in per-size results while still revealing a band of competitive sizes.


### **Syntactic Chunking**

Syntactic (aka language‑aware) chunking uses language heuristics to split at logical code boundaries. Specifically, the syntactic chunking used in these tests is based on LangChain’s `RecursiveCharacterTextSplitter.from_language` to split each file, falling back on fixed_token when the file type is not supported. LangChain defines a priority of language‑specific separators (functions/classes/blocks) and snaps boundaries to them. Note: `chunk_overlap` is not reliably enforced by this splitter in practice (boundaries snap to separators and merges vary); empirical checks across multiple languages and sizes confirmed this behavior. AST‑based chunking was not implemented for these trials.

Our hypothesis was that by preserving logical code units (functions, methods, classes) and providing semantically complete snippets, the syntactic chunking method would lead to improved accuracy compared to the equivalent fixed-size chunking tests, as chunks would not be split mid‑function/class.
See Results and Analysis for how this compared in practice.

### **Packed Chunking**

Packed chunking combines whole files until the token budget is efficiently filled, avoiding mixes of partial + whole files within a single chunk. It will only split a file when that single file is larger than `chunk_size`; in that case packed uses the syntactic splitter to break that file. `packed_fixed` follows the same boundary rule but splits oversized files using the fixed_token splitter. We added `packed_fixed` after fixed_token showed more predictable overlap and slightly stronger results in some cases. Packed avoids wasting space and proved to be a strong overall performer (see Results and Analysis for metrics).

Originally this wasn't intended to be used as a real chunker but instead to understand diminishing returns at higher chunk sizes. As testing continued it became clear that this chunking strategy can offer significant benefit to precision with a trade‑off of slightly lower recall. As the packed context grows beyond smaller-file sizes in a dataset, the model appears to make fewer, more accurate guesses. See Results and Analysis for performance outcomes.

---

## **Methodology: Testing with fraim**

To evaluate each chunking method, we integrated them into fraim and ran them across four datasets:

* **Generated**: Small, artificial projects with one vulnerability each. These act as sanity checks rather than decision drivers.  
* **Validation Benchmarks**: 30 results across 20 files; structured XBEN cases with longer spans.  
* **Juice Shop**: Large, real-world vulnerable application (594+ source files overall); baseline includes 75 results across 32 files.  
* **Verademo**: Moderately sized real-world repo; baseline includes 65 results across 19 files.

We measured:

* **True Positives (TP)** – correct detections  
* **False Positives (FP)** – incorrect detections  
* **False Negatives (FN)** – missed vulnerabilities  
* **Precision** – proportion of correct detections among all detections  
* **Recall** – proportion of real vulnerabilities successfully found  
* **F1-Score** – harmonic mean of precision and recall  
* **Processing Time** – average total processing time per configuration

Comparator summary and limitations: Matching requires the same vulnerability `type` and at least one overlapping file, then selects the best semantic match based on message/explanation text. False negatives fully inside unscanned spans are excluded from recall. Because benchmarks scan only baseline‑referenced files, precision here reflects noise within vulnerable files and will differ from full‑repo precision.

Labels and units:
- Labels: `project` = whole-repo input; `file` = one file per chunk; `original_*` = line-based splits; `syntactic_*` = language-aware splits; `fixed_token_*` = token-capped; `packed_*` = packs whole files when possible.
- Units: Chunk sizes are in tokens for all methods except `original`, which uses lines.
- Skipped Lines metric: Reported as `Skipped Lines / 100k`. This normalizes the number of unscanned lines to a per‑100,000‑lines scale so results are comparable across repos of different sizes. Skipped lines are excluded from recall (misses fully inside unscanned spans are not counted against recall) but provide a coverage signal to interpret recall alongside.
- Precision scope: Because only baseline-referenced files are scanned, precision reflects noise within vulnerable files; full‑repo precision may differ when unrelated files are scanned.

---

## **Results and Analysis**

Note on very large contexts: We plan to revisit the largest chunk sizes after a fix to skipped‑lines handling lands in a future fraim version. Some high‑size configurations may shift once that change is in place.

### **Method-Specific Insights**

- Project: Competitive accuracy/speed trade-off versus large packed sizes. On validation-benchmarks it achieves the highest precision (0.6923) but with lower recall; performance varies with repo scale and clustering. When to use: small/medium repos where speed matters and memory permits whole‑project or near‑project inputs.
- Packed: Best average F1 at mid sizes (≈5.5–7k). Larger sizes increase precision but reduce recall; efficient use of token budget by packing whole files. When to use: strong default (start around ~5.5–7k tokens); prefer larger sizes when precision is prioritized over recall.
- Fixed-size: Results do not form a smooth curve across sizes; as chunk sizes change, critical context falls in/out of boundaries and many files are smaller than large sizes, yielding step-changes and plateaus. A competitive band emerges around mid sizes. When to use: benchmarking to probe size sensitivity; very large single files when semantic splitting is unavailable.
- Syntactic (language-aware): Tracked fixed-size and sometimes underperformed slightly. Likely because hypothesis generation can rely on non‑structured signals (names, comments) even when functions split. Note: requested `chunk_overlap` is not reliably enforced by the splitter, making syntactic less predictable. Fixed_token is slightly more predictable because overlap is applied deterministically. When to use: when preserving function/class boundaries is desirable and language splitters are accurate enough; expect parity with fixed‑token otherwise.
- Triage impact: With triage enabled, approximate hypotheses often become true positives, increasing recall relative to these no‑triage benchmarks.

### **Method Highlights (Current Results)**

- Average across datasets:
  - Best F1: `packed 5.5k` at 0.3348
  - Best Precision: `packed_fixed 25k` at 0.5495
  - Best Recall: `packed_fixed 1k` at 0.3875
  - Speed: `packed 6.5k` is fastest at 93.06s average; `project` runs in 192.25s; `file` in 197.55s
  - Large‑budget trade‑off: large packed/packed_fixed sizes boost precision (up to 0.5495 with `packed_fixed 25k`) while recall and F1 vary by dataset.

- repos/validation-benchmarks:
  - Best F1: `packed 5.5k` at 0.4675
  - Best Precision: `project` at 0.6923; trade-off with low recall 0.1765
  - Best Recall: `syntactic 3k` at 0.5385

- repos/juice-shop:
  - Best F1: `fixed_token 20k` at 0.2687
  - Best Precision: `packed_fixed 25k` at 0.5
  - Fastest: `packed_fixed 20k` at 99.50s

- repos/verademo:
  - Best F1: `packed_fixed 13k` at 0.3363
  - Best Precision: `packed 30k` at 0.6471
  - Best Recall: `packed_fixed 4k` at 0.4038
  - Fastest: `packed_fixed 15k` at 77.94s

### **Dataset Behavior**

* **Generated**: Nearly perfect results since vulnerabilities were simple and isolated—useful as a sanity check but not used to guide conclusions.  
* **Validation Benchmarks**: Best F1 around \~5.5k tokens (packed); larger sizes increased precision but not F1, and recall dropped as unrelated cases were combined—evidence of context dilution (question 1).  
* **Juice Shop**: The hardest dataset. Cross-file vulnerabilities made small chunks unreliable (question 2), while large chunks mixed test and production code, creating “context valleys” that confused the model (question 1).  
* **Verademo**: Showed similar trends to Juice Shop but less extreme.

### **Cross-Method Insights**

- **Whole project/module**: Competitive with large packed sizes on average and a good speed/accuracy trade‑off on some datasets; performance varies on larger repos.  
- **Whole files**: Strong baseline for simple repos, weaker for complex ones.  
- **Fixed-size**: Showed the expected recall–precision trade-off, highlighting the balance between the two questions.  
- **Syntactic (language-aware)**: Tracked fixed-size closely, underperforming slightly in some cases; less predictable than fixed_token due to overlap not always being enforced by the splitter.  
- **Packed (+ packed_fixed variant)**: Best overall balance of accuracy, efficiency, and cost. `packed ~5.5–7k` leads on average and on validation‑benchmarks; `packed_fixed ~13k` edges out on verademo. At very small budgets, `packed_fixed` tends to show better recall; differences mainly appear when large files must be split before packing (syntactic vs. fixed_token bases).

### **What We Expected vs. What We Found**

- Project vs. Packed: Expected whole-project context to be too noisy except on tiny repos. Found project compares closely with large packed sizes while often running faster (see Method Highlights), making it a pragmatic accuracy/speed trade‑off on some datasets.
- Fixed-size “sweet spot”: Expected a smooth size→accuracy curve. Found F1 varies non‑smoothly as critical context falls in/out of chunk boundaries and many files are smaller than larger sizes; a competitive band emerges around mid sizes.
- Syntactic (language-aware) vs. Fixed-size: Expected boundary‑preserving splits to outperform fixed-size. Found syntactic tracks fixed-size and sometimes underperforms slightly; likely because hypothesis generation can rely on non‑structured cues (names/comments), and requested `chunk_overlap` isn’t reliably enforced by the splitter.
- Context extremes: Expected too much context to dilute recall and too little to miss signals. Found this pattern across datasets: larger packed sizes increase precision but reduce recall; smaller sizes reverse the trade‑off (numbers in Method Highlights).

---

## **Conclusion: The Future of Context in AI Security**

Our experiments confirm both sides of the tension:

* **Yes, LLMs lose accuracy when fed too much code**. Beyond a certain point, additional context dilutes the signal and reduces recall.  
* **Yes, LLMs also miss vulnerabilities without enough context**. Small snippets alone often don’t provide the information needed to recognize flaws.

False positives from small chunks can be filtered out during triage, but false negatives from insufficient context are lost forever. This makes recall-critical scenarios especially sensitive to under-chunking.

The best default today is **packed chunking**, with sizes around 5.5–7k tokens performing strongly across datasets and larger sizes (e.g., 25–30k packed_fixed) delivering the highest precision on average. That said, the **project** strategy now compares closely with large packed configurations while running faster in some cases, making it a pragmatic option when accuracy and speed both matter. The bigger lesson is that **context quality matters more than size**. More tokens aren’t always better if they introduce irrelevant code.

The future likely lies in adaptive chunking—strategies that dynamically choose context size based on repository scale, file structure, and the vulnerabilities being sought. This way, both questions can be balanced intelligently rather than forced into a one‑size‑fits‑all approach.

---

## **Model and Reproducibility**

Benchmarks used the default model `gemini/gemini-2.5-flash`. To reproduce the results and regenerate graphs:

- Install deps and set your API key: `uv sync --dev` and export `GEMINI_API_KEY` (or `GOOGLE_API_KEY`).
- Run benchmarks (defaults from the CLI): `uv run refraim benchmark run --times 3`
- Optionally limit datasets: `--datasets repos/juice-shop repos/verademo generated/*`
- Render graphs and summaries: `uv run refraim benchmark graph --output-dir blog/results`

Limitations reminder: These benchmarks scan only baseline‑referenced vulnerable files with triage disabled; precision reflects within‑file noise, recall excludes unscanned spans, and matching weights semantic message/explanation similarity.
