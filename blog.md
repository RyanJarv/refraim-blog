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

## **Results and Analysis**

Numbers reflect fraim v0.5.1 and mirror `results/results.md` and `results/*.html`.

Note on very large contexts: We plan to revisit the largest chunk sizes after a fix to skipped‑lines handling lands in a future fraim version. Some higher chunk‑size configurations may shift once that change is in place.

Caveats (read with the numbers)
- Scan scope: only baseline‑referenced vulnerable files; precision reflects within‑file noise.
- Skipped lines: spans excluded from recall; reported separately; fix is planned.
- Matching: same vuln type + overlapping file; best semantic match on text/snippet.
- Triage: disabled in these benchmarks; enabling it typically converts some approximate hypotheses into true positives, improving recall in practice.

Method Labels Guide
- `project` (whole repo), `module` (package group), `file` (per‑file), `original_*` (line‑based), `syntactic_*` (language‑aware), `fixed_token_*` (deterministic overlap), `packed` (packs whole files; splits oversized with syntactic), `packed_fixed` (splits oversized with fixed_token).

### Overall Best

| Goal | Recommendation | Why | Trade‑off |
| --- | --- | --- | --- |
| Best overall balance | `packed 5.5–7k` | Best avg F1 and solid speed | Mid‑size only |
| Highest precision | `project` | Fewest false positives on small/medium repos; no skipped‑lines | Memory/cost; repo must fit |
| Highest recall | `packed_fixed 1–4k` | More hits found | Noisier; triage recommended |
| Fastest overall | `packed 6.5k` | Lowest avg time (~93.06s) | Dataset dependent |
| Small/medium repo speed | `project` | Whole‑repo context | Memory/cost limits |
| Very large single files | `fixed_token` base | Predictable overlap | Structure not preserved |


Performance Dashboard (visual idea)
- Option A: scatter (F1 vs Processing Time) highlighting `packed 5.5k` and `project`.
- Option B: grouped bars per dataset for Best F1, Best Precision, Best Recall.

Visuals legend: “Right/up is better; larger bubble = tokens; color = method; hollow markers = recall‑first picks.”

### Best By Dataset

| Dataset | Best F1 | Best Precision | Best Recall | Fastest | Recommendation |
| --- | --- | --- | --- | --- | --- |
| Average Across Datasets | `packed 5.5k` (0.3348) | `packed_fixed 25k` (0.5495) | `packed_fixed 1k` (0.3913) | `packed 6.5k` (93.06s) | Default pick |
| Validation Benchmarks | `packed 5.5k` (ties `packed 23k`) at 0.4675 | `project` (0.6923) | `syntactic 3k` (0.5385) | `packed 5.5k` (55.44s) | Prefer packed mid‑sizes; project for precision |
| Juice Shop | `packed_fixed 12k` (0.2933) | `packed_fixed 25k` (0.5) | `fixed_token 1k` (0.2766) | `packed_fixed 20k` (99.50s) | Use packed_fixed mid/large; avoid extremes |
| Verademo | `packed_fixed 13k` (0.3363) | `packed 30k` (0.6471) | `packed_fixed 4k` (0.4151) | `packed_fixed 15k` (77.94s) | Start at packed_fixed ~13k |
| Generated | `packed 3k` (0.8) | `packed 3k` (0.6667) | `project` (1.0) | `original 0.7k` (16.16s) | Sanity check; don’t over‑weight |

<!-- INSERT: Performance Dashboard (Average Across Datasets) — inline HTML from results -->
<!-- INSERT: Interactive Report — results/average.html -->

<!-- INSERT: Interactive Report — results/repos.validation-benchmarks.html -->

<!-- INSERT: Interactive Report — results/repos.juice-shop.html -->

<!-- INSERT: Interactive Report — results/repos.verademo.html -->

Note on precision picks: While some datasets show highest precision at very large packed sizes (e.g., 25–30k), we are not recommending those configurations until a skipped‑lines handling fix lands. Prefer `project` for precision on small/medium repos; retest large packed sizes after the fix.

### About The Results

With the winners and graphs in view, here’s why these patterns emerge and how to apply them.

Note: These benchmarks reflect the hypothesis‑generation stage (possible vulnerabilities) with triage disabled; in real runs, triage confirms/deduplicates and often converts approximate hypotheses into true positives, improving recall.

* The signal vs. noise trade‑off

  Larger contexts tend to raise precision but lower recall: with more surrounding code, the model has better clues to reject false alarms, but relevant signals can be diluted. Smaller contexts do the opposite: recall rises as more candidates get flagged, but precision drops. The mid‑sized packed sweet spot (≈5.5–7k) preserves enough local relationships without overwhelming the model, which is why it wins on average.

* Cross‑file flows and boundary alignment

  Many vulnerabilities span functions and files (e.g., user input source → transformations → sink). Very small chunks sever these paths and hurt recall. Very large chunks mix unrelated modules, tests, and vendor code, blunting recall by burying cues. Packing entire small files together keeps adjacency that helps the model follow flows without dragging in unrelated code.

* Boundary reflow and critical context

  Changing chunk sizes can reflow a file across calls and accidentally split key cues. Example: a 90‑line file at 30‑line chunks yields [1–30], [31–60], [61–90]; bumping to 45‑line chunks yields [1–45], [46–90]. If the critical evidence sits around lines 31–60, the larger setting now straddles two calls, lowering recall even though the chunk grew. This boundary effect explains some of the “non‑monotonic” wiggles you see when F1 doesn’t steadily improve with size.

* Why packed works well

  Packed keeps files whole and fills the window efficiently, only splitting when necessary. This maintains coherent, human‑like review units (files and their immediate neighbors). As more relevant code is packed together (beyond single‑file limits), the model tends to make fewer—but more accurate—hypotheses, boosting precision while holding F1 steady.

* Why project can be fast (and competitive)

  On small/medium repos, whole‑repo context creates natural proximity between related code without chunking overhead, often running faster with accuracy comparable to large packed sizes. The trade‑off is memory/cost; it doesn’t scale well on big repos.

* Why fixed‑token helps with very large single files
  When a single file exceeds the window, deterministic overlap can help preserve context across boundaries, improving recall over naïve splits. It’s a practical fallback for monolith files where preserving cross‑boundary flow matters more than perfect syntactic boundaries.
  This particularly applies to non‑structured signals in the code (complexity, naming, comments, “code smell”), which helps explain why fixed‑token sometimes outperformed syntactic chunking.

* Why syntactic didn’t stand out (from earlier exploration)

  Two factors likely contributed:
  - Overlap reliability: the language‑aware splitter we used (LangChain’s RecursiveCharacterTextSplitter) doesn’t reliably enforce overlap, so cues can be split across boundaries.
  - Non‑structured cues drive early hypotheses: at this stage we aren’t confirming vulnerabilities, we’re generating plausible candidates to follow up in triage. LLMs lean on signals that aren’t tightly bound to AST units—function and variable names, inline comments, docstrings/documentation, and general “code‑smell” (e.g., undue complexity, awkward patterns, inconsistent naming, missing comments). This mirrors how human reviewers start with threat‑model and intent rather than deep control‑flow. Because these cues span or ignore strict syntax boundaries, preserving whole files and predictable overlap often matters more than perfect syntactic splits. In our runs, that helps explain why syntactic tracked fixed‑size and sometimes underperformed.

### Practical takeaways

* Balanced default: use `packed 5.5–7k`.
* Precision‑first: use `project` (when the repo fits); we are not recommending very large packed sizes until the skipped‑lines fix lands.
* Recall‑first: consider `1–4k` chunks and lean on triage later.
* Huge single file: use fixed‑token overlap to avoid severing key flows.


## **Conclusion: The Future of Context in AI Security**

Our results point to a simple truth: the right context beats more context. Mid‑sized packed windows around 5.5–7k tokens consistently balance accuracy and speed, while very large inputs raise measured precision but often hurt recall through dilution and “lost‑in‑the‑middle.” Tiny chunks miss the bigger picture. While on small or medium repositories, a project‑wide view can match large packed runs and often finish faster. Interestingly, syntactic splitting did not outperform fixed‑size with more predictable overlap; keeping files intact with predictable overlap helped more than perfect boundaries. 

Going forward, it seems likely that best approach when it comes to more advanced chunkers, is using both a hybrid and adaptive approach. Tracking important unstructured-data like the projects threat-model, important comments, and the purpose or intent of the code. For structured data, AST parsing can then allow building a simple map of the code—imports, calls, and shared names—and packs nearby, related pieces together so the model sees the evidence that belongs together without extra noise. Tracking how this data is related together may also give us additional insights on what effect specific information has on the results.

With a better understanding of how various datasets are affected, we may be able to adjust the approach at runtime depending on the results of profiling the code. With our current results this might be: project‑like packing for small repos, mid‑size packed windows for larger ones, and fixed‑overlap for monolithic files. While this may not make significant impact at the moment, it seems likely the differences will become more apparent as this process is optimized. When the model is unsure about a finding, it may expand context locally along the code graph instead of making chunk sizes bigger.

In the shorter term, we will make packed ~5.5–7k the default strategy in fraim. After the skipped‑lines issue is fixed we can retest the higher precision options seen in these results, potentially allowing for the user to prefer either recall or precision. Regardless of what options are chosen later on these tests will serve to help determining the ideal size and splitting methods for fraim.