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
* How does different sized chunks affect chunking strategies?


It's important to note that in all the tests below, only vulnerable files were scanned, this means that the whole project scan does not include the project [README.md](http://README.md) or any code that is believed is unrelated to the vulnerabilities present.

### **How The Tests Work**

* Datasets and baselines: Each suite provides a `code/` tree plus an `expected.sarif` baseline that encodes the ground truth. Baselines follow SARIF 2.1.0 and list findings with relative file paths, line regions, and a standardized vulnerability `type` in `properties`.
* What we scan: For every chunking configuration, fraim scans only files referenced by the baseline (the known vulnerable files). Triage is disabled to measure raw detection performance of the chunking strategy and model.
* Actual vs. expected: The scan emits SARIF; results are matched to baseline findings by file and vulnerability type with a semantic similarity check on messages/explanations. Matches become true positives; unmatched actuals are false positives; unmatched expected are false negatives.
* Skipped code: If the scanner can’t parse parts of a file, those line ranges are recorded as “unscanned.” False negatives entirely inside those spans don’t count against recall. Charts report this as “Skipped Lines * 100,000” for scale.
* Metrics and aggregation: Precision, recall, and F1 are computed per run and then aggregated across repeats. Tables and charts in this section summarize those aggregated outcomes for each dataset and configuration.
* Reading the labels: `project` means whole-repo input; `file` means one file per chunk; `original_*` are small fixed-size chunks; `syntactic_*` uses language-aware splits; `fixed_token_*` caps tokens per chunk; `packed_*` fills chunks by combining small files without splitting them.
* Interpreting precision: Because only baseline-referenced files are scanned, precision here reflects noise within vulnerable files. Full-repo precision may differ when scanning unrelated files; recall trends are still indicative of context sufficiency.


For each test and dataset we measured the following statistics:

*Recall – Of all the real vulnerabilities present, what percentage did we find?*  
*Precision – Of all the vulnerabilities the scanner reported, what percentage were actually real?*  
*F1-Score – The harmonic mean of Precision and Recall. Provides a single, balanced score of the scanner's accuracy.*

### **Project (Baseline)**

This strategy attempts to add the entire project in a single call or combines files to fill the LLMs context window, it does not chunk the code but instead cacatenates files while maintaining file path metadata. We originally predicted this would degrade performance overall for most code bases, while potentially leading to high accuracy in smaller ones as the whole project can be included without using many tokens. 

Based on our tests we did not see any benefit of using this strategy across any of the tested datasets. As expected we saw degrated recall, with precision matching other methods on smaller datasets. It seems likely that this strategy never outperformed any other strategies due chunk sizing being large enough that the packed tests fully contained the smaller datasets in the same way the this project chunking strategy did, resulting in no practical benefit to using the project chunking strategy.

At the moment, we don't believe there is any benefit to using the project strategy. ... (TODO: is more testing needed?)

### **File**

The file strategy does not perform chunking or concatenation but instead attempts to fit the entire file in a single call by default, however, if the file is larger then the LLM's max context window it would be broken across several calls similar using the same method as the fixed-size strategy. We originally hypothesized that the performance will vary depending on file size and internal code organization and that it may perform adequately for small self-contained files but fail to provide appropriate context while struggling with large files or vulnerabilities requiring cross-file context.

Compared to project, the file strategy suffered in precision but performed better in recall and as expected, we saw better results on the validation-benchmarks dataset which contains many unrelated sub-projects. When looking at the f1-score on other datasets, performance was relatively poor, with file having the worse f1-score compared to other strategies.

### **Fixed-Size Chunking**

Fixed-size chunking splits code into calls of a set number of tokens without combining multiple files. For fixed-size, the chunk size to use becomes an important question due to the tradeoff off the performance hit we saw in the project strategy and our expectation that vulnerability identification requires sufficient context. 

Our hypothesis here was that testing this method across a range of chunk sizes will show a "sweet spot" where accuracy is maximized and false positives are minimized. Too small, and critical context is lost; too large, and noise increases, or token limits are hit, leading to a curved chunk size/detection graph forming a parabola that reveals the optimal chunk size for a LLM.

The results however, where slightly more complicated then we expected, the f1-score did not form a parabola but instead appeared to be somewhat random. This makes sense when we consider what happens when critical context falls in and out of chunks as chunk size changes along with the fact that chunk sizes larger then the analyzed file size will not show measurable differences in the results. So, larger chunk sizes may show improved performance in practice but only when most of the analyzed files can benefit from it, combined with the critical context issue, the chunk sizes that may normally result in measurable differences appear somewhat random.

The issue of critical context falling in and out of chunks can be seen in other tests as well. This makes sense when you consider increasing chunk sizes can affect later API calls; for example, if a 90 line file is analyzed in 3 API calls the chunks may be lines 1-30, 31-60, and 61-90 and when analyzed with an increased chunk size it changes to two API calls with chunks containing lines 1-45 and 46-90. When the critical context required to hypothesize about a given vulnerability exists on lines 31-60, reducing the chunks will break up the required context across two API calls resulting in degraded performance despite the ideal chunk size being larger then 30 lines. This behavior can be seen best in the generated tests where fixed_token_10000_1000 and fixed_token_20000_2000 which had worse recall then both lower and higher chunk sizes and performed worse the test between the two, fixed_token_13000_1300.


### **Syntactic Chunking**

Semantic chunking uses syntactic parsing based on the current language to split at logical code boundaries. Specifically, the syntactic chunking used in these tests is based on langchain's RecursiveCharacterTextSplitter.from_language functionality to split each file, falling back on fixed_token when the file type is not supported. Langchain's RecursiveCharacterTextSplitter works by defining a priority of identifiers for the analyzed language, it attempts to include entire top level namespaces like functions and classes when possible falling back on language specific namespaces and control flow statements when necessary.

TODO: Run a tests to verify exactly how langchains language aware splitter works.

Our hypothesis was that by preserving logical code units (functions, methods, classes) and providing semantically complete snippets, the syntactic chunking method will lead to improved accuracy compared to the equivalent Fixed-Size chunking tests as the chunks will not be split in the middle of a function, class, or method.

Surprisingly, the results differed with our hypothesis and syntactic chunking performed very similarly to fixed-size, and in some cases performed slightly worse then fixed-size chunking. Further testing here is needed to validate why this occured, but I believe this behavior can be explained by considering what exactly we are benchmarking, the nature of code review, and the reasoning for using syntactic chunking.

The critical context issue covered in the Fixed-Size chunking results is the is the same reason we speculated that syntactic chunking will outperform fixed-size chunking, however, what is considered essential for hypothesizing about vulnerabilities may not be as closely tied to the syntactic structure of the code as we previously assumed. This makes a bit more sense when we consider that we are not looking for confirmed vulnerabilities but instead possible vulnerabilities that will be followed up on later, in this context, function and variable names, code comments and documentation may be the primarily method that the LLM is hypothesizing about vulnerabilities. Considering the process of manual code review, the initial step in identifying vulnerabilities is to understand the threat model which does not involve in-depth understanding of the code but instead it's purpose and how it relates to broader contexts, the application as a whole, the environment it's ran in, and the expectations of the users. While this broader context wasn't included in our testing, it can still be inferred by the non-structured info in the chunked code blocks. 

Additionally, the non-structured info in this chunked-code can contribute significantly to what a human reviewer might refer to as the very vague concept of code-smell, high-risk areas that are overly complex, use awkward coding patterns, confusing naming, lack comments, or any other of the many signals you can find in an arbitrary chunk of code. These tend to be areas that you want to follow up on as a code reviewer, and in our case, the accuracy of the LLMs inclination to follow up on real vulnerabilities is what we are benchmarking.

While what I described above seems to be what is occuring, futher testing is needed to confirm this updated hypothesis is correct.

### **Packed Chunking**

Packed chunking combines small files until the context window is efficiently filled, without splitting files mid-way. This chunking strategy was added after realizing the average file size of the dataset may affect the results in a non-obvious way, since the other chunkers won't fill the context window when ran against smaller files. The effect this has on the results is that as the chunk size grows, it's noticable result on the results will diminish and become less obvious, even in the case that the LLM benefits from having additional context. This avoids wasting space on under-filled contexts and proved to be the strongest performer overall. The **packed\_7000\_700**configuration increased F1 scores by \~28% and precision by \~83% compared to the baseline, making it the most practical default strategy. 

Originally this wasn't intended to be used as a real chunker but instead to understand the effects of the diminishing returns at higher chunk sizes. As testing continued it became clear that this chunking strategy can offer significant benefit to precision with a trade off of slightly lower recall. Based on this it seems that as additional code was added to the context window, beyond the size of smaller files in the dataset, the LLM appeared to be able to make more accurate guesses filtering out one's that it did not consider applied given it's additional understanding of the application. Basically, as the packed context grew, less, but more accurate guesses where made.

Packed chunking produced the best overall f1-score indicating that this tradeoff is worth it if you consider precision and recall equally. However, despite the fact that diminishing recall is the most likely cause of close-enough guesses, these close-enough guesses are still valid due to triager's tendancy to only need approximatatly correct inputs. The triage step's purpose is to filter out incorect vulnerabilities, however, it also has the side effect of finding the correct vulnerability given an approximate guess. This can be seen in practice by running fraim with triage on, the recall results end up being higher then what we show here.

---

## **Methodology: Testing with fraim**

To evaluate each chunking method, we integrated them into fraim and ran them across four datasets:

* **Generated**: Small, artificial projects with one vulnerability each.  
* **Validation Benchmarks**: 100+ structured test cases (XBEN-001+).  
* **Juice Shop**: A large, real-world vulnerable application (594+ files).  
* **Verademo**: A moderately sized real-world repo.

We measured:

* **True Positives (TP)** – correct detections  
* **False Positives (FP)** – incorrect detections  
* **False Negatives (FN)** – missed vulnerabilities  
* **Precision** – proportion of correct detections among all detections  
* **Recall** – proportion of real vulnerabilities successfully found  
* **F1-Score** – harmonic mean of precision and recall  
* **Processing Time** – average runtime per chunk

---

## **Results and Analysis**

### **Dataset Behavior**

* **Generated**: Nearly perfect results, since vulnerabilities were simple and isolated—showing that minimal context can be enough (question 2).  
* **Validation Benchmarks**: Performed best around \~25k tokens. Beyond that, recall dropped as unrelated test cases were combined—clear evidence of context dilution (question 1).  
* **Juice Shop**: The hardest dataset. Cross-file vulnerabilities made small chunks unreliable (question 2), while large chunks mixed test and production code, creating “context valleys” that confused the model (question 1).  
* **Verademo**: Showed similar trends to Juice Shop but less extreme.

### **Cross-Method Insights**

* **Whole project/module**: Too noisy and expensive to be practical.  
* **Whole files**: Strong baseline for simple repos, weak for complex ones.  
* **Fixed-size**: Showed the expected recall–precision trade-off, highlighting the balance between the two questions.  
* **Semantic**: Underperformed, leaving an open question about why.  
* **Packed**: Best balance of accuracy, efficiency, and cost.

---

## **Conclusion: The Future of Context in AI Security**

Our experiments confirm both sides of the tension:

* **Yes, LLMs lose accuracy when fed too much code**. Beyond a certain point, additional context dilutes the signal and reduces recall.  
* **Yes, LLMs also miss vulnerabilities without enough context**. Small snippets alone often don’t provide the information needed to recognize flaws.

False positives from small chunks can be filtered out during triage, but false negatives from insufficient context are lost forever. This makes recall-critical scenarios especially sensitive to under-chunking.

The best default today is **packed chunking**, particularly the packed\_7000\_700 configuration, which improves precision dramatically while keeping F1-scores stable. But the bigger lesson is that **context quality matters more than size**. More tokens aren’t always better if they introduce irrelevant code.

The future likely lies in adaptive chunking—strategies that dynamically choose context size based on repository scale, file structure, and the vulnerabilities being sought. This way, both questions can be balanced intelligently rather than forced into a one-size-fits-all approach.
