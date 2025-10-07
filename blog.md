# **Optimizing LLM Context for Vulnerability Scanning**

## **Introduction: The Challenge of Code Context in AI Security**

When it comes to vulnerability detection with LLM's how context is used plays an important role in performance. The chunking strategy, the size you choose, along with several other factors, directly determine what the model can see and reason about. In this post we will explore the effects chunking strategies and context size has when using Language Models (LLMs) to discover vulnerabilities OWASP top 10 vulnerabilities using fraim. 

Previously to this research our understanding was that LLMs need broad context to understand potential vulnerabilities, however, in our personal experience, providing too much code in the context can often overwhelm the model and degrade its accuracy. With this post we hope to answer whether LLM's tend to miss vulnerabilities if they are fed too many lines of code at once as well, the effect different strategies have on peformance, as how much surrounding code the LLM needs to identify a vulnerability in the first place, specifically in the context of fraim.

Two opposing forces shape accuracy in practice. Provide too much code and relevant signals get diluted—e.g., mixing tests and production files or many unrelated modules reduced recall in our experiments. Provide too little and the model misses necessary cues—e.g., a sink defined in one file and the input validation in another, or a subtle configuration that disables sanitization. This trade‑off also affects speed and cost: larger prompts are slower and more expensive, while smaller prompts risk false negatives.

Our experiments benchmarked chunking strategies to find the balance: enough context to hypothesize vulnerabilities reliably without overwhelming the model or budgets. The results below quantify how different strategies and sizes shift precision, recall, runtime, and overall F1.

### TL; DR

- Default: start with `packed 5.5–7k` for balanced F1 and speed.
- Precision‑first: try `packed(_fixed) 13–25k` (expect lower recall).
- Speed‑first on small/medium repos: consider `project` if memory permits.
- Very large single files: prefer `fixed_token` for predictable overlap.

## **Our Approach: The AI-Powered Vulnerability Scanner**

Our system, **fraim**, uses a two-stage workflow. First, it chunks repositories into smaller inputs so an LLM can scan them for vulnerabilities. Then, a **triager stage** validates those findings with full repository access, filtering out false alarms.

The scanner’s first stage produces candidate findings—hypotheses based on chunked context. The triage stage does not use chunking; it evaluates candidates with full‑repository context to confirm, deduplicate, and discard false positives. In these benchmarks we disable triage to isolate the effects of chunking strategy and size. Because real runs include triage, you should expect better end‑to‑end results in practice than the raw figures shown here.

The scanner targets common vulnerabilities such as SQL injection, XSS, and CSRF across multiple languages (Python, JavaScript, Java, C, etc.). In these cases, context is critical. A snippet that looks unsafe in isolation may be harmless in its actual usage—or vice versa. This is why finding the right chunking method is so important.

## **The Mechanics of Code Chunking for LLMs**

Chunking means splitting a large codebase into smaller segments that fit within an LLM’s token window. It’s not only about size constraints; it’s about **feeding the model the right information**.

The key idea is to maximize useful context while minimizing irrelevant details. Done poorly, chunking can cut off the information the model needs to reason about a function. Done well, it lets the model focus on vulnerabilities in context without distraction.

### Why Context Matters: Two Small Examples

```javascript
// file: routes.js
app.get('/item', (req, res) => {
  const id = req.query.id;
  // looks safe alone; the sink is elsewhere
  res.send(renderItem(id));
});

// file: utils.js
export function renderItem(id) {
  // vulnerable: concatenation into HTML; upstream lacks sanitization
  return `<div>${id}</div>`;
}
```

```python
# file: db.py
def find_user(db, username):
    # appears risky in isolation but upstream enforces a strict allowlist
    return db.execute(f"SELECT * FROM users WHERE name = '{username}'")

# file: api.py
def handle_request(req):
    username = req.args['u']
    if username not in ALLOWLIST:  # context that prevents exploitation
        return 403
    return find_user(DB, username)
```

Isolated chunks like this can miss the cross‑file risk in the first example; overly large chunks can bury the allowlist guard in the second. Additionally, you may need to consider that all chunks prior to a given vulnerability will affect how that vulnerability is layed out in relation to it's containing chunk. Increasing sizes just slightly can push important context over onto into a seperate API call.

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

This strategy attempts to add the entire project in a single call or combines files to fill the LLM’s context window. In practice, it packs whole files up to about 70% of the model’s max context and only splits a single file that exceeds the budget using the fixed_token splitter (MaxContextChunker on a fixed_token base). We originally predicted this would degrade performance overall for most codebases, while potentially performing well on small repositories. With the latest runs, project shows a competitive accuracy/speed trade‑off against large packed sizes; see Results and Analysis for metrics.

Label: `project`

### **File**

The file strategy does not perform concatenation but instead attempts to fit the entire file in a single call by default; however, if the file is very large, it is split across several calls using the same fixed_token approach sized to about 70% of the model’s max context to avoid near‑limit failures. We originally hypothesized that the performance will vary depending on file size and internal code organization and that it may perform adequately for small self-contained files but fail to provide appropriate context while struggling with large files or vulnerabilities requiring cross-file context.

Compared to project, the file strategy suffered in precision but performed better in recall and, as expected, did better on the Validation Benchmarks dataset which contains many unrelated sub‑projects. When looking at the F1‑score on other datasets, performance was relatively poor, with file generally below the stronger strategies.

Label: `file`

### **Fixed-Size Chunking**

Fixed-size chunking splits code into calls of a set number of tokens without combining multiple files. For fixed-size, the chunk size to use becomes an important question due to the trade-off between larger contexts (potential dilution) and providing enough context to hypothesize a vulnerability.

Our hypothesis here was that testing this method across a range of chunk sizes would surface a “sweet spot” where accuracy is maximized and false positives minimized. Too small, and critical context is lost; too large, and noise increases or token limits are hit. In practice, the F1-score does not trace a smooth curve because critical context can fall in and out of chunk boundaries (as sizes change as mentioned in the  [The Mechanics of Code Chunking for LLMs](#the-mechanics-of-code-chunking-for-llms) section), and because many files are smaller than the larger chunk sizes. This produces apparent “randomness” in per-size results while still revealing a band of competitive sizes.

Label: `fixed_token_*` (e.g., `fixed_token 5.5k`)


### **Syntactic Chunking**

Syntactic (aka language‑aware) chunking uses language heuristics to split at logical code boundaries. Specifically, the syntactic chunking used in these tests is based on LangChain’s `RecursiveCharacterTextSplitter.from_language` to split each file, falling back on fixed_token when the file type is not supported. LangChain defines a priority of language‑specific separators (functions/classes/blocks) and snaps boundaries to them. Note: `chunk_overlap` is not reliably enforced by this splitter in practice (boundaries snap to separators and merges vary); empirical checks across multiple languages and sizes confirmed this behavior.

Our hypothesis was that by preserving logical code units (functions, methods, classes) and providing semantically complete snippets, the syntactic chunking method would lead to improved accuracy compared to the equivalent fixed-size chunking tests, as chunks would not be split mid‑function/class.

See Results and Analysis for how this compared in practice.


### **Packed Chunking**

Packed chunking combines whole files until the token budget is efficiently filled, avoiding mixes of partial + whole files within a single chunk. It will only split a file when that single file is larger than `chunk_size`; in that case packed uses the syntactic splitter to break that file. `packed_fixed` follows the same boundary rule but splits oversized files using the fixed_token splitter. We added `packed_fixed` after fixed_token showed more predictable overlap and slightly stronger results in some cases. Packed avoids wasting space and proved to be a strong overall performer (see Results and Analysis for metrics).

Originally this wasn't intended to be used as a real chunker but instead to understand diminishing returns at higher chunk sizes. As testing continued it became clear that this chunking strategy can offer significant benefit to precision with a trade‑off of slightly lower recall. As the packed context grows beyond smaller-file sizes in a dataset, the model appears to make fewer, more accurate guesses. See Results and Analysis for performance outcomes.

Labels: `packed_*` and `packed_fixed_*`

### **Original (Line‑Based)**

The original strategy is the legacy, line‑based splitter. It caps chunks by line count rather than tokens and does not attempt to preserve language structure. It remains useful when tokenization is unavailable or when you need a very simple, deterministic window.

Sizes for all other chunking methods are defined in tokens while the original strategy uses lines.

Label: `original_*`

## **Methodology: Testing with fraim**

To evaluate each chunking method, we integrated them into fraim and ran them across four datasets:

* **Generated**: Small, artificial projects with one vulnerability each. These primarily act as somewhat simple sanity checks rather than decision drivers. The results from these datasets are not included in the overall average. 
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

Comparator summary and limitations: Matching requires the same vulnerability `type` and at least one overlapping file, then selects the a match based on a combination of semantically matching the vulnerability description and code embeddings of the vulnerable snippet.

Un-Scanned Lines Correction: False negatives fully inside unscanned spans are excluded from recall to mitigate any noticable performance drop in the results, this is done to make it possible to understand the expected results when this issue is fixed in a future version of fraim.

Precision scope: Because benchmarks scan only baseline‑referenced files, precision here reflects noise within vulnerable files and will differ from full‑repo precision which may contain additional context and information critical to understanding the threat-model.

## **Results and Analysis**

Numbers reflect fraim v0.5.1 and mirror `results/results.md` and `results/*.html`.

Note on very large contexts: We plan to revisit the largest chunk sizes after a fix to skipped‑lines handling lands in a future fraim version. Some higher chunk‑size configurations may shift once that change is in place.

Key Finding
- Mid‑sized packed context (5.5–7k tokens) delivers the best balance of accuracy and speed on average (best F1 0.3348). Larger contexts increase precision but reduce recall.

TL;DR
- Default: use `packed 5.5–7k` for best average F1 and strong speed.
- Precision‑first: use `packed_fixed 25–30k` (accept lower recall).
- Recall‑first: use `packed_fixed 1–4k` (accept lower precision; triage recommended).
- Speed on small/medium repos: consider `project` when memory permits.

Caveats (read with the numbers)
- Scan scope: only baseline‑referenced vulnerable files; precision reflects within‑file noise.
- Skipped lines: spans excluded from recall; reported separately; fix is planned.
- Matching: same vuln type + overlapping file; best semantic match on text/snippet.

Method Labels Guide
- `project` (whole repo), `module` (package group), `file` (per‑file), `original_*` (line‑based), `syntactic_*` (language‑aware), `fixed_token_*` (deterministic overlap), `packed` (packs whole files; splits oversized with syntactic), `packed_fixed` (splits oversized with fixed_token).

Quick Picks (Goal → Recommendation → Why → Caveat)
- Best average F1 → `packed 5.5–7k` → Balanced precision/recall → Mid‑size only.
- Highest precision → `packed_fixed 25–30k` → Fewer false positives → Lower recall.
- Highest recall → `packed_fixed 1–4k` → More hits → Noisier; triage later.
- Fastest overall → `packed 6.5k` → Lowest average time (≈93.06s) → Dataset‑dependent.
- Small/medium repo speed → `project` → Whole‑repo context → Memory/cost trade‑off.
- Very large single file → `fixed_token` base → Deterministic overlap → Weaker structure.

Decision Flow (text)
- Is speed critical? → Yes → `project`; No → Continue
- Very large single file? → Yes → `fixed_token` base; No → Continue
- Need max precision? → Yes → `packed_fixed 25–30k`; No → Default: `packed 5.5–7k`

Performance Dashboard (visual idea)
- Option A: scatter (F1 vs Processing Time) highlighting `packed 5.5k` and `project`.
- Option B: grouped bars per dataset for Best F1, Best Precision, Best Recall.

Visuals legend: “Right/up is better; larger bubble = tokens; color = method; hollow markers = recall‑first picks.”

### At‑a‑Glance Summary Table

| Dataset | Best F1 | Best Precision | Best Recall | Fastest | Recommendation |
| --- | --- | --- | --- | --- | --- |
| Average Across Datasets | `packed 5.5k` (0.3348) | `packed_fixed 25k` (0.5495) | `packed_fixed 1k` (0.3913) | `packed 6.5k` (93.06s) | Default pick |
| Validation Benchmarks | `packed 5.5k` (ties `packed 23k`) at 0.4675 | `project` (0.6923) | `syntactic 3k` (0.5385) | `packed 5.5k` (55.44s) | Prefer packed mid‑sizes; project for precision |
| Juice Shop | `packed_fixed 12k` (0.2933) | `packed_fixed 25k` (0.5) | `fixed_token 1k` (0.2766) | `packed_fixed 20k` (99.50s) | Use packed_fixed mid/large; avoid extremes |
| Verademo | `packed_fixed 13k` (0.3363) | `packed 30k` (0.6471) | `packed_fixed 4k` (0.4151) | `packed_fixed 15k` (77.94s) | Start at packed_fixed ~13k |
| Generated | `packed 3k` (0.8) | `packed 3k` (0.6667) | `project` (1.0) | `original 0.7k` (16.16s) | Sanity check; don’t over‑weight |

Reports
- Average Across Datasets: `results/average.html`
- Validation Benchmarks: `results/repos.validation-benchmarks.html`
- Juice Shop: `results/repos.juice-shop.html`
- Verademo: `results/repos.verademo.html`
- Generated: `results/generated.html`

If your viewer supports inline HTML, you can also explore the interactive reports below. Otherwise, use the report links above.

<!-- INSERT: Performance Dashboard (Average Across Datasets) — inline HTML from results -->
<!-- INSERT: Interactive Report — results/average.html -->

<!-- INSERT: Interactive Report — results/repos.validation-benchmarks.html -->

<!-- INSERT: Interactive Report — results/repos.juice-shop.html -->

<!-- INSERT: Interactive Report — results/repos.verademo.html -->

### Strategy Comparison

- Packed (recommended default)
  - Sweet spot: 5.5–7k (F1 0.3348). Trade‑off: 25–30k boosts precision (~0.55) with lower recall.
  - When to use: start here for most codebases.
- Project (speed‑optimized on small/medium repos)
  - Performance: compares closely to large packed; often faster (≈192.25s avg).
  - When to use: speed matters and memory permits.
- Fixed‑token base (fallback for very large single files)
  - Strength: deterministic overlap; predictable splits when one file exceeds budget.
  - When to use: very large single files or when syntactic splitting is unreliable.
- Syntactic (language‑aware)
  - Note: tracks fixed‑size; can underperform slightly; overlap not always enforced.
  - When to use: prefer boundary preservation when splitters are accurate.

### Key Patterns (why these results matter)

- Larger contexts raise precision but reduce recall; smaller contexts invert that trade‑off.
- `packed 5.5–7k` is the best default on average; `packed_fixed ~13k` edges Verademo.
- `packed_fixed 25–30k` yields the highest precision on average; expect lower recall.
- `project` compares with large `packed` on some datasets and often runs faster.
- For single files exceeding the budget, prefer `fixed_token` for predictable overlap.

### Dataset Notes

- Validation Benchmarks: best F1 ties `packed 5.5k` and `packed 23k` (0.4675); precision rises with size; recall falls.
- Juice Shop: cross‑file vulnerabilities made small chunks unreliable; very large contexts mixed tests and prod, diluting recall.
- Verademo: mirrors Juice Shop trends; `packed_fixed 13k` edges best F1.
- Generated: near‑perfect by construction—good sanity check but not used to guide conclusions.

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

Benchmarks used the default model `gemini/gemini-2.5-flash`.

> Warning: Benchmarks with `--times 3` issue multiple API calls and can incur non‑trivial cost at larger chunk sizes. Consider rate limits and estimate spend as `calls × tokens × price_per_1k`.

To reproduce the results and regenerate graphs:

- Install deps and set your API key:

  ```bash
  uv sync --dev
  export GEMINI_API_KEY=... # or GOOGLE_API_KEY
  ```

- Run benchmarks (3 runs per configuration by default):

  ```bash
  uv run refraim benchmark run --times 3
  ```

- Optionally limit datasets:

  ```bash
  uv run refraim benchmark run --times 3 --datasets repos/juice-shop repos/verademo generated/*
  ```

- Render graphs and summaries (paths are relative to your current working directory):

  ```bash
  # From repo root
  uv run refraim benchmark graph --output-dir blog/results

  # From inside blog/
  uv run refraim benchmark graph --output-dir results
  ```

Notes
- Results reflect fraim v0.5.1; numbers mirror `results/results.md` and `results/*.html`.
- Processing times are environment‑dependent and serve as relative comparisons.
- Triage was disabled here; enabling it typically converts some approximate hypotheses into true positives, improving recall in practice.
- Precision reflects within‑file noise because only baseline‑referenced files are scanned; full‑repo precision can differ.

Limitations reminder: These benchmarks scan only baseline‑referenced vulnerable files with triage disabled; precision reflects within‑file noise, recall excludes unscanned spans, and matching weights semantic message/explanation similarity.
