# Optimizing LLM Context for Code Analysis

## **Summary**

This post explores the challenge of providing Large Language Models (LLMs) with the right amount of code context for effective security analysis. While LLMs need broad context to understand potential vulnerabilities, in our experience, providing too much code in the context can overwhelm the model and degrade its accuracy. Conversely, providing too little context results in missed detections.

To confirm our assumption regarding large context degradation, we benchmark file concatenation with whole projects, as well as individual modules. Testing on single files with no concatenation or chunking provides our baseline to compare other results. Then to find the optimal balance, we investigate including **whole files** in the context, as well as breaking files into **fixed-size chunks** and **semantic chunks** (based on functions and classes) to determine how each method impacts performance. By measuring metrics like detection rates, false positives, and processing speed, we aim to identify the most effective chunking approach that maximizes accuracy while minimizing noise, paving the way for more reliable AI-powered vulnerability scanning.

## **Audience**

The audience will likely be security professionals, developers, and perhaps other AI/ML practitioners. Avoid overly academic jargon where simpler terms suffice, but don't shy away from technical details where necessary.	

## **Overview**

## **1\. Introduction: The Challenge of Code Context in AI Security**

* **Description:**  
  * Briefly introduce the growing role of AI in vulnerability scanning and the inherent challenge of providing sufficient, relevant code context to Large Language Models (LLMs) without overwhelming them.  
  * Explain the challenges faced with code analysis, context win  
  * Highlight the trade-offs involved (e.g., speed, accuracy, cost).  
  * Introduce the context problem in code analysis for AIs.  
* **Simplified Draft:**  
  * AI accuracy is closely tied to how effectively it ‘understands’ the underlying code.  
  * To reliably understand the behavior of a snippet it’s necessary to consider it in the broader context of its containing code, and in some cases the codebase as a whole.  
  * In fraim, we implemented a two step process that first uses code chunking to reduce context window, and a final verification stage with tool access to explore the entire repository with standard tracing tools.  
  * The question we hope to identify in this blog post is what is the ideal method of code chunking that balances providing both relevant code and limits the context to avoid LLM degradation. How much of a difference does each code chunking make? Additionally what does the graph of chunk size performance look like?

### **2\. Our Approach: The AI-Powered Vulnerability Scanner**

* **Description:**  
  * Introduce fraim and its current capabilities and describe how it uses chunking and the triager process.  
  * Specify the types of vulnerabilities the scanner looks for and the file types it processes, as provided in the ai\_scanner\_overview.  
  * Go into more detail about why context is particularly important for Code Analysis and even more for Vulnerability Analysis, include examples.  
* **Simplified Draft:**  
  * The fraim vulnerability scanner employs chunking to dissect repositories for individual analysis, followed by a triager process that has access to the whole repo to validate findings.  
  * Our scanner currently targets a range of common vulnerabilities, including SQL Injection, XSS, CSRF, and others, across various programming languages such as Python, C, Java, and JavaScript.  
  * TODO: Write up an overview with examples on why context is important.

### **3\. The Mechanics of Code Chunking for LLMs**

* **Description:**  
  * Explain what code chunking is in the context of LLM-based analysis.  
  * Discuss why chunking is necessary (e.g., token limits, improved focus, reduced noise).  
  * Briefly touch upon the current state of chunking in the AI scanner.  
* **Simplified Draft:**  
  * Code chunking is the process of breaking down a large codebase into smaller, manageable segments for an LLM to process.  
  * This isn't just about fitting within token limits; it's about optimizing the LLM's focus, ensuring it receives the most relevant information while minimizing distractions.  
  * TODO: Briefly touch upon the current state of chunking in the AI scanner.

### **4\. Exploring Chunking Strategies: A Deep Dive**

* **Description:**  
  * Detail various chunking strategies, including the ones identified in the discussion.  
  * For each strategy, explain its approach, potential benefits, and drawbacks.  
  * This is where you'll introduce the concept of:  
    * feeding whole files as a baseline  
    * combining files to represent multiple files in the context  
    * fixed-size chunking  
    * syntactic chunking.  
  * Once we understand the behaviors of different approaches to chunking we may want to choose what to use depending on the file type.  
* **Simplified Draft:**  
  * Our research covers several prominent chunking methodologies, starting with the foundational question: what if we didn't chunk at all?  
  * Then we follow up with what happens when we start combining files?  
  * Beyond this baseline, we're rigorously testing fixed-size chunks of varying lengths and exploring the nuances of syntactic chunking, which aims to preserve logical code structures.

#### **Sub-Section Y: Context Scope – Whole Project**

* **Description:**  
  * Explain the concept of feeding an entire project to the LLM by concatenating all files, and how this is the opposite of chunking.  
  * Discuss its theoretical pros (complete context) and very practical cons (token limits, performance, cost, noise).  
* **Hypothesis:**   
  * This approach will likely fail due to token limits for most real-world projects. For very small projects, it might offer the most comprehensive context, potentially leading to high accuracy for small projects and high noise and false positives for larger ones due to overwhelming the LLM.  
* **Notes:**  
  * Why do we want to test this?  
  * We don’t expect it to work well, but it is added here for completeness as well as a base line to compare with the sub-component and file level chunking.  
  * The first part of this test will be checking the number of tokens of concatenating everything, if it’s over the context length then we can rule it out as an option.  
  * See simonw/ttok  
* **When to Use:**   
  * This approach is best reserved for **very small, self-contained projects** where the total token count is well within the LLM's context window. It serves primarily as a **theoretical baseline** to understand the upper limits of providing context, rather than a practical, scalable solution for real-world applications.  
* **Simplified Draft:**  
  * While seemingly ideal for providing full context, feeding the entire project to an LLM quickly runs into insurmountable challenges related to token limits and computational expense.  
  * This approach is actually the opposite of chunking because we need to combine files and teach the LLM how to separate them.  
  * This approach isn’t practical for medium to large projects but may be an option for smaller ones.  
  * We can measure the number of tokens in a project by combining files with `simonw/files-to-prompt` then measuring tokens with `simonw/ttok`.  
* **Results:**  
  * Accuracy/Detection Rate: Does the LLM miss more with larger/smaller chunks, or with different chunking methods?

#### **Sub-Section X: Context Scope – Project Module**

* **Description:**  
  * Explain what we consider a module here (a group of files serving a similar functional purpose).  
  * Covers the motivation for testing this.  
* **Hypothesis:**   
  * This approach will likely perform poorly with its significant context use. For very small modules, it might offer the most comprehensive context, potentially leading to high accuracy for small projects and high noise and false positives for larger ones due to overwhelming the LLM. q  
* **When to Use:**   
  * Use this strategy when analyzing vulnerabilities that are likely to **span multiple, closely related files** that form a distinct functional unit (e.g., a user authentication service, a data processing pipeline). It's a good middle-ground when single-file analysis is insufficient, but whole-project analysis is infeasible.  
* **Simplified Draft:**  
  * The name may differ between coding languages but in general a module is a group of files that functionally serve a similar purpose and some portion of the code is intended to only be called from within that same module.  
  * The idea behind this approach is that because functionality is grouped by module we can get most of the benefits of the whole project test while considerably reducing token count.

  #### **Sub-section A: Baseline – Whole Files (No Chunking)**

* **Notes:**  
  * This is the only test that does not chunk or combine files, processing each file individually.  
  * It likely makes sense to consider this the control group for your benchmarking.  
* **Hypothesis:**   
  * This will serve as our control. We hypothesize that its performance will vary depending on file size and internal code organization. It may perform adequately for small, self-contained files but fail to provide appropriate context while also likely struggle with very large files or vulnerabilities requiring cross-file context.  
* **When to Use:**   
  * This should be the **default starting point or control group** for analysis. It is most effective for codebases where functionality is well-encapsulated within single files or for identifying vulnerabilities that do not depend on cross-file interactions. It's simple and provides a clear baseline for comparison.  
* **Simplified Draft:**  
  * Individual files are a bit more reasonable then what we’ve tested so far and still represent similar functionality, however this can be inconsistent.  
  * The length of individual files can vary a lot and there are many different methods of organizing code, so we may see more varying results for this across different code bases.  
  * Since this is the only test that does not combine or chunk files we’ll be using it as our baseline.

#### **Sub-section B: Fixed-Size Chunking**

* **Description:**  
  * Detail how fixed-size chunking works (e.g., splitting by a certain number of lines or characters).  
  * Discuss its simplicity and the critical question of optimal chunk size. Introduce the idea of a "chunk size/detection curve."  
* **Hypothesis:**   
  * This method will show a "sweet spot" for chunk size where accuracy is maximized and false positives are minimized. Too small, and critical context is lost; too large, and noise increases, or token limits are hit, leading to a curved chunk size/detection graph forming a Parabola that reveals the optimal chunk size for a LLM.  
* **When to Use:**   
  * This method is useful when dealing with **very large files that exceed token limits** or when semantic parsing is too complex or slow for a given language. It's a straightforward way to break down monolithic files, especially when you are in the process of **benchmarking to find an optimal context size** for your specific LLM and use case.  
* **Simplified Draft:**  
  * Fixed-size chunking offers a straightforward approach, segmenting code into predefined blocks.  
  * From this testing we hope to understand how varying chunk lengths impact the LLM's ability to identify vulnerabilities.  
  * From this testing we hope to learn what effects chunk size has on identifying vulnerabilities, as well as what the ideal chunk size is.

#### **Sub-section C: Syntactic Chunking**

* **Description:**  
  * Explain syntactic chunking, specifically mentioning function, method, and class boundaries using language‑aware splitters (e.g., LangChain). Note: current implementation is not AST‑based and requested `chunk_overlap` is not reliably enforced by the splitter.  
  * Highlight the hypothesis that preserving functions/classes/methods might lead to better understanding and fewer missed vulnerabilities  
* **Hypothesis:**   
  * By preserving logical code units (functions, methods, classes) and providing syntactically complete snippets, this method will lead to improved accuracy compared to equivalent Fixed-Size Chunking test as the chunks will not be split in the middle of a function, class, or method.  
* **When to Use:**   
  * This is the preferred strategy when **maintaining the logical integrity of the code is paramount**. It is particularly effective for object-oriented or highly structured code, as it ensures that the LLM receives complete, executable units (like functions or classes) for analysis, reducing the risk of misinterpretation caused by arbitrary splits.  
* **Simplified Draft:**  
  * Syntactic chunking leverages language specific syntax to identify logical boundaries within the code, such as functions, methods, and classes.  
  * It’s difficult to understand the behavior of a function when you only have half of it.  
* **Actual Results**  
  * Syntactic chunking provided no benefits over fixed-sized and original.  
  * Based on this we can assume that hypothesizing about vulnerabilities has little relation to the code structure, and that performance is largely based on the non‑structured info found in a code base.  
    * This makes sense when you consider how code is reviewed manually: you don’t start with deep reading of control‑flow; threat model and “code smell” guide early analysis. In practice, function/variable names, comments, docstrings, and general code‑quality signals (complexity, awkward patterns, inconsistent naming, missing comments) appear to drive hypothesis quality more than strict AST boundaries.

### **5\. Methodology: Testing Chunking Strategies with refraim**

* **Description:**  
  * Outline the proposed testing process.  
  * Explain how refraim will be used to evaluate each chunking method.  
  * Mention metrics that will be used (found in the [Results and Analysis](#6.-results-and-analysis)) and briefly explain their significance for a vulnerability scanner.  
* **Simplified Draft:**  
  * To evaluate these chunking strategies, we will integrate each chunker with our fraim system and test them using refraim.  
  * Our methodology will focus on quantifiable metrics such as vulnerability detection rates (maximizing true positives), false positive ratios (minimizing noise for analysts), and the overall processing efficiency for each chunking approach.

### **6\. Results and Analysis** {#6.-results-and-analysis}

* **Questions:**
    * Accuracy/Detection Rate: Does the LLM miss more with larger/smaller chunks, or with different chunking methods?
        * Does the LLM miss vulns if it is fed too many lines of code at once? Current wisdom is that LLMs with large context windows lose specificity if you actually use that entire context window.
        * How much surrounding code does an LLM need to spot the vuln in the first place?  (Our current code scanning workflow doesn't make tools available in the triage step. False positives in step one are ok. Triage can use tools to filter them out, at additional time/cost. But false negatives are missed for good.)
    * False Positives: Does a particular chunking method lead to more incorrect detections?
        * How does total reported findings affect precision? How does it affect recall?
    * Performance/Speed: How much longer does it take to process a codebase with one chunking method vs. another?
        * How does chunking size affect the processing time?
        * Is processing time related to performance?
    * Dataset traits: How do same‑file clustering vs. cross‑file dependencies shift optimal sizes and favored strategies across datasets?
* **Description:**  
  * This section will present the actual findings from benchmarking.  
  * It will discuss how each chunking strategy performed across the chosen metrics.  
  * Include examples of how different chunking strategies either helped or hindered the detection of a specific vulnerability.  
* **Test Metrics:**  
  * **True Positives (TP):** The number of known, real vulnerabilities that the scanner correctly identified. This is your primary measure of success.  
  * **False Positives (FP):** The number of times the scanner reported a vulnerability that doesn't actually exist. A high FP rate creates noise and analyst fatigue.  
  * **False Negatives (FN):** The number of known, real vulnerabilities that the scanner *missed*. This is a critical failure metric.  
  * **Detection Rate (Recall):** Calculated as `TP / (TP + FN)`. This tells you, "Of all the real vulnerabilities present, what percentage did we find?" This is a key metric for a security tool.  
  * **Precision:** Calculated as `TP / (TP + FP)`. This tells you, "Of all the vulnerabilities the scanner reported, what percentage were actually real?" This measures the signal-to-noise ratio.  
  * **F1-Score:** The harmonic mean of Precision and Recall (`2 * (Precision * Recall) / (Precision + Recall)`). This provides a single, balanced score of the scanner's accuracy  
  * **Processing Time:** The average chunk processing time.  
  * **Skipped Lines / 100k:** The number of unscanned lines normalized per 100,000 lines to compare across repositories of different sizes. Misses fully inside unscanned spans are excluded from recall, so this metric provides important coverage context alongside recall.
* **Methodology Assumptions:**
  * Benchmarks scan only baseline‑referenced vulnerable files; non‑referenced files are excluded.
  * Triage is disabled to surface chunking effects; precision therefore reflects noise within vulnerable files and differs from full‑repo precision.
  * Comparator matches on vulnerability `type` and at least one overlapping file, then uses semantic similarity over message/explanation; each expected can be matched at most once.
* **Units Policy:**
  * Chunk sizes are measured in tokens for all strategies except `original`, which uses lines. Labels and figures should reflect native units.
* **Simplified Draft:**  
  * TODO: Come back to this when testing is finished.

### **7\. Conclusion: The Future of Context in AI Security**

* **Description:**  
  * Summarize the key findings and their implications for developing more effective AI-powered vulnerability scanners.  
  * Discuss future research directions based on your results. What's next? Are there other chunking strategies to explore (e.g., recursive analysis chunking, dependency-aware chunking, call graph-based chunking)? How could the current strategies be refined?  
  * Briefly mention the tools you're using if relevant to the chunking process itself.  
* **Simplified Draft:**  
  * TODO: Come back to this when testing is finished.

 
