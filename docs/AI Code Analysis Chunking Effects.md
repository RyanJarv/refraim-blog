

# **The Granular Codebase: Strategic Chunking and Size Optimization for High-Performance AI Code Analysis Pipelines**

## **I. Strategic Imperative: Why Granularity Defines Code Analysis Success**

The application of Large Language Models (LLMs) to complex software engineering tasks, such as code analysis, generation, and vulnerability detection, necessitates rigorous context management. While LLMs excel at understanding and generating text, their utility in enterprise-scale codebases is fundamentally constrained by limitations in processing vast, information-dense inputs. Strategic chunking, the process of dividing large code repositories into optimally sized and structured segments, is not merely a preprocessing step; it is the critical engineering layer that determines the performance, cost-efficiency, and ultimate reliability of the AI analysis pipeline.

### **A. Contextual Limitations of Large Language Models (LLMs)**

The core constraint of any LLM is the context window, which defines the maximum amount of input, measured in tokens, that the model can process or "remember" at any given time.1 Although modern models boast significantly extended context windows—with some systems like Google's Gemini featuring up to 1-million tokens 2—relying solely on window expansion is insufficient and economically unsound. Processing inputs of such monumental size inherently requires increased computational power, leading directly to higher running costs and latency in downstream responses.1 For large organizations that incur charges based on token usage, merely passing an entire codebase or large documentation set through the context window is often economically prohibitive.4 The context window, therefore, functions as a cost multiplier, making Retrieval-Augmented Generation (RAG) a necessary cost-control and context-optimization strategy to avoid computation being "wasted" on simple data retrieval.4  
Furthermore, LLMs exhibit cognitive limitations when processing extremely long sequences, even when the data fits within the allowed token limit. This deficiency manifests as the "lost-in-the-middle" problem, where relevant information buried deep within a long document is frequently overlooked by the model.5 Empirical data confirms this degradation, illustrating that LLM performance follows a sigmoid or exponential decay curve as reasoning complexity increases. Performance degradation also intensifies as context length increases, even within the same difficulty level.6 This context bloat actively degrades accuracy, necessitating a shift in focus from maximizing the size of the input to maximizing the *information density* of the input.7 Strategic chunking is the engineered methodology used to select the smallest possible set of high-signal tokens that maximize the likelihood of achieving the desired analysis outcome.7

### **B. Retrieval-Augmented Generation (RAG) in Code Analysis**

Retrieval-Augmented Generation is the established technique for overcoming LLM context limits by dynamically fetching relevant external knowledge, such as code snippets or documentation, and augmenting the prompt.8 Chunking serves as the foundational first stage of the RAG pipeline, transforming the source material into manageable units for embedding and efficient vector search.10  
Code, unlike natural prose, presents a unique challenge due to its highly structured, context-dependent nature. An effective code chunk must balance two contradictory requirements: it must be small enough to allow for performant vector search and fit within the embedding model's context limits 5, yet simultaneously large enough to maintain meaningful, self-contained semantic information, such as a complete function or class definition.5 The success of the RAG pipeline hinges entirely on resolving this fundamental trade-off.

## **II. Taxonomy of Chunking Strategies in Code Analysis**

The selection of a chunking strategy is arguably the single most critical architectural decision in building a high-performance AI code analysis pipeline. Generic text chunking methods often fail catastrophically when applied to source code because they disregard the inherent syntactic structure.

### **A. Naive and Character-Based Methods**

The simplest strategies, while fast and easy to implement, prove inadequate for complex code analysis.

#### **Fixed-Size and Recursive Chunking**

Fixed-size chunking (character or token count) is the most crude method, segmenting text into predefined lengths irrespective of content or logical structure.12 While overlapping consecutive chunks can be used to mitigate abrupt breaks and maintain some minimal semantic continuity 13, this approach fundamentally disregards the logical boundaries of code. Recursive chunking introduces a hierarchy of separators (e.g., splitting by paragraphs, then sentences, then characters) to attempt to honor structure iteratively.12

#### **Limitations for Code**

Naive chunking methods struggle profoundly with structured content like source code.10 They frequently split meaningful syntactic units—such as class definitions, function bodies, or control structures—across two or more chunks. This fragmentation results in chunks that are syntactically incomplete and, consequently, semantically incoherent.10 The structural integrity of the code is destroyed, which severely degrades the downstream retrieval accuracy.

### **B. Structural and Syntax-Aware Chunking (The Code-Native Approach)**

High-performance code analysis requires structural chunking, where the segmentation boundaries align perfectly with the logical organization of the programming language constructs.

#### **Function and Class-Level Segmentation**

The most basic, yet essential, structural approach for code files involves segmenting the data by functions or classes.10 Chunking by these units ensures that each segment represents a complete, logical unit of execution or encapsulation.15 This maximal contextual cohesion within the chunk is vital for tasks like code generation, summarization, or bug localization, as the LLM requires the full definition and scope of the method to operate correctly. A common piece of advice is explicitly not to split up a function or class definition.14

#### **Abstract Syntax Tree (AST)-Based Chunking**

Abstract Syntax Trees (ASTs) represent the hierarchical structure of source code, making them the ideal basis for structural segmentation.10 AST-based chunking splits code precisely at meaningful boundaries derived from the syntax tree, such as specific function definitions, loop bodies, or conditional statements.17 This ensures that every resulting segment remains syntactically valid.  
AST-based chunking fundamentally surpasses token-level or character-level methods by preserving the hierarchical structure and semantic relationships that are essential for deep code understanding.16 For code analysis, retrieval is a semantic task, but the unit of retrieval must be syntactic. The strong empirical performance observed with AST-based methods confirms that syntactic validity is highly correlated with semantic coherence in code. This makes AST-based approaches, such as cAST (Chunking via Abstract Syntax Trees), the established baseline for overcoming the context destruction caused by naive splitting.10 cAST is recognized as an effective, structure-aware strategy that significantly enhances retrieval performance in sophisticated code RAG workflows.10

### **C. Semantic and Hybrid Chunking**

Beyond strict structure, advanced methods incorporate meaning or multiple contextual dimensions. Semantic chunking identifies chunk boundaries where the thematic or topical meaning shifts.5 This is achieved by generating vector embeddings for sequential groups of sentences and detecting breakpoints where the semantic distance between consecutive groups exceeds a threshold.5  
Hybrid chunking combines these dimensions, such as integrating structural boundaries with semantic similarity.18 For code, a hybrid approach might utilize ASTs to define potential chunks, then use semantic distance to group related functions or classes, potentially even across files, to maximize conceptual cohesion.10

## **III. The Critical Dimension of Chunk Size: Accuracy vs. Context**

Once a chunking *strategy* is selected (structural being preferred for code), the determination of the optimal *size* becomes a complex optimization problem involving a direct trade-off that profoundly impacts the LLM’s eventual output quality.

### **A. The Trade-off Model: Precision vs. Context Retention**

#### **Small Chunks (High Precision, Low Context)**

Smaller chunks result in vector embeddings that are highly focused on a specific piece of information. This granularity leads to high retrieval precision, as the embedding closely matches targeted queries.19 However, the primary risk associated with very small chunks is the loss of crucial surrounding context. If a retrieved chunk is a snippet of code, it may be semantically useless without the necessary defining scope, variable initialization, or function signature found outside its boundaries.19 This leads to semantic fragmentation.20

#### **Large Chunks (High Context, Low Precision)**

Larger chunks excel at context preservation, containing the full scope of a logical unit, potentially encompassing an entire file or a very large class.19 The risk here is the introduction of extraneous noise, or irrelevant text, which inevitably degrades the quality of the vector search match.6 Large context chunks are more likely to contain irrelevant information that acts as a distraction to the LLM, increasing the confusion factor and the propensity for hallucinations.3

### **B. Empirical Insights on Optimal Size**

Empirical evaluation suggests that extreme sizes rarely yield optimal results. Testing across various datasets indicates that very small chunks (e.g., 128 tokens) and very large chunks (e.g., 2,048 tokens) generally underperform compared to medium-sized or structurally derived segments.22 The optimal size is highly dependent on the nature of the content and the task required of the LLM.20  
For code, the principle is that the optimal size often aligns with the natural structural boundary. However, the objective for code analysis is finding the *minimum effective size* that retains semantic coherence for the specific task, not simply the largest size that fits the context window. This approach mitigates the cost, latency, and performance degradation associated with large inputs.3  
The optimal size decision is strongly influenced by the analysis objective:

1. **High-Context Tasks (e.g., Code Generation or Refactoring):** These require the full scope of a function or class definition, making function/class-level structural chunks optimal.  
2. **High-Precision Tasks (e.g., Vulnerability Detection):** Analysis focusing on pinpointing defects benefits from finer granularity. Research demonstrates that using code-chunk-based segmentation dramatically improved both the F1-score (a 42.0% gain) and accuracy (a 53.9% gain) over using the full function body for vulnerability prediction.23 This implies that for highly specific defect analysis, the optimal chunk is a structural *sub-unit* within a function, requiring specialized micro-chunking to maximize the signal-to-noise ratio.

The complexity of selecting an optimal size necessitates iterative testing and empirical evaluation against query sets.5 The following table summarizes the typical trade-offs observed in code analysis RAG pipelines:  
Table I: Chunk Size Trade-offs in Code RAG

| Chunk Size Category | Primary Retrieval Effect | Risk/Trade-off | Optimal Use Case in Code RAG | Supporting Evidence |
| :---- | :---- | :---- | :---- | :---- |
| Very Small (e.g., \< 256 tokens) | High Precision, Fast Embeddings | Context Loss, Fragmented Semantics | Targeted Keyword/Fact Retrieval, Indexing Child Chunks | 19 |
| Optimized Structural (Function/Class) | Structural Coherence, High Recall | Size Variability, Internal Noise | Core Code Generation, Semantic Search, Summarization | 10 |
| Large (e.g., \> 2048 tokens) | Maximum Context Preservation (File level) | Noise/Distraction, High Cost, Lost-in-the-Middle | Summarization, High-Level Architecture Analysis | 5 |

## **IV. Advanced Architectures for Context Retrieval**

The limitations of relying on a single, fixed chunk size are overcome through advanced, multi-layered retrieval architectures that dynamically adapt context granularity to the user query.

### **A. Hierarchical Retrieval: The Parent-Child Solution**

Parent-Child chunking is a state-of-the-art RAG technique specifically designed to resolve the inherent conflict between the high precision offered by small chunks and the rich, complete context provided by large chunks.25

#### **Mechanism and Application in Code**

This strategy establishes two levels of context granularity 25:

1. **Child Chunks (Indexing):** These are smaller, highly focused structural segments (e.g., 100–500 tokens), such as code blocks or specific AST nodes. These segments are embedded and indexed, maximizing the precision of the vector search matching against the user query.24  
2. **Parent Chunks (Retrieval):** These are larger segments (e.g., 500–2000 tokens) representing the complete, coherent context, typically the entire function or class definition containing the child chunk. These parent chunks are stored but not indexed directly.24

When a query is executed, the system searches the precise child chunk embeddings. Upon finding a match, the system retrieves and supplies the associated larger parent chunk to the LLM for final generation.26 This dual mechanism is ideally suited for code, where a child chunk might represent a code block, and the parent guarantees the full functional scope and definition. This architectural approach avoids retrieval noise, minimizes hallucinations, and maintains token efficiency during the costly generation phase.24  
The implementation of Parent-Child retrieval often leverages AST-based recursive splitting.10 The parent chunk is established by traversing the AST to fit large nodes into coherent units, and the child chunks are generated from these parents. This maximizes structural integrity while simultaneously optimizing search precision, offering adaptive granularity for simple factual lookups versus complex architectural analysis.25

### **B. Context Management for Repository-Level Analysis**

When analysis shifts from individual files to repository-level tasks, such as generating architectural components or managing complex, cross-file dependencies 27, traditional vector search over static chunks becomes inefficient.

#### **Agentic Context Management (Non-RAG Approach)**

Some cutting-edge solutions, exemplified by systems like Claude Code, fundamentally reject traditional RAG methodologies. Instead of statically pre-chunking the entire codebase into a vector store, these systems leverage an LLM-native search combined with external low-level tools such as ripgrep, jq, and bash commands.7  
This revolutionary approach avoids the complexities associated with similarity function confusion, chunking decisions, and reranking complexities inherent in traditional RAG.29 The LLM acts as an agent, utilizing external organization and indexing systems (the file system) to read files incrementally and retrieve relevant information dynamically, akin to a human developer using search tools.7 This active context management strategy efficiently bypasses the static limitations of chunking for dynamic, large-scale architectural queries, ensuring that context bloat is minimized through tool-based, self-managed task lists.7

## **V. Quantitative Performance Analysis and Benchmarking**

Empirical validation is paramount to determine the efficacy of advanced chunking strategies and size decisions. Data confirms that structure-aware chunking delivers significant and measurable performance improvements over generic segmentation.

### **A. Metrics for Code Analysis RAG Pipelines**

Performance evaluation in code RAG pipelines typically relies on several key metrics:

1. **Retrieval Metrics:** During the retrieval phase, performance is measured using **Recall@k** (the proportion of relevant source code segments found within the top *k* retrieved results) and **Precision** (the accuracy of the retrieved segments).10  
2. **Task-Specific Metrics:** For specific downstream tasks like vulnerability detection or fault localization, classification metrics such as the **F1 score** (harmonic mean of precision and recall) and **accuracy** are used to measure effectiveness.23

### **B. Comparative Performance of Structural Chunking**

Quantitative studies involving structural chunking methods confirm their superiority for code data.

#### **cAST Performance Gains**

The structure-aware cAST strategy demonstrates reliable gains across key code retrieval benchmarks.10 The results show that aligning chunks with abstract syntax boundaries reliably supplies richer and more accurate evidence to the LLM compared to baseline methods.10

* **Code-to-Code Retrieval (RepoEval):** Structural awareness led to gains of **1.2–3.3 points in Precision** and **1.8–4.3 points in Recall**.10  
* **Natural Language (NL)-to-Code Retrieval (SWE-Bench):** Gains were smaller but still significant, registering **0.5–1.4 points in Precision** and **0.7–1.1 points in Recall**.10

The observation that gains are greater in code-to-code retrieval than in NL-to-code retrieval suggests that while structural context is vital for understanding and manipulating code semantics, the task of translating complex human intent (NL query) into the correct code unit is inherently harder, and thus structural chunking provides less independent leverage.

#### **Vulnerability Detection Optimization (FuncVul)**

Perhaps the most compelling evidence for task-specific chunk size optimization comes from analysis pipelines designed for defect detection. In vulnerability prediction tasks using the FuncVul model, employing code-chunk-based segmentation, rather than using the full function as the unit of analysis, resulted in a dramatic **53.9% accuracy gain** and a **42.0% F1-score improvement**.23 This reinforces the principle that for high-precision analytical tasks, the optimal chunk size is often strategically smaller than the natural structural boundary (the entire function), specifically tailored to maximize the signal-to-noise ratio relevant to the defect being sought.  
Table II: Comparative Performance of Chunking Strategies in Code Analysis

| Chunking Strategy | Primary Code RAG Task | Observed Performance Metric | Quantitative Gain Over Baseline | Source |
| :---- | :---- | :---- | :---- | :---- |
| Fixed-Size (Baseline) | Retrieval/Generation | Precision, Recall | Low/Inconsistent, Risk of Context Loss | 10 |
| AST-Based (cAST) | Code-to-Code Retrieval (RepoEval) | Precision/Recall | Up to 3.3 (P) / 4.3 (R) points increase | 10 |
| AST-Based (cAST) | NL-to-Code Retrieval (SWE-Bench) | Precision/Recall | Up to 1.4 (P) / 1.1 (R) points increase | 10 |
| Code-Chunk-Based (FuncVul) | Vulnerability Detection | Accuracy / F1 Score | 53.9% Accuracy / 42.0% F1 Score increase | 23 |

## **VI. Operational and Economic Considerations**

Chunking decisions carry significant implications beyond just retrieval accuracy, directly shaping the operational cost, latency, and scalability of AI code analysis pipelines.

### **A. Cost Modeling: Indexing vs. Inference**

A RAG pipeline involves two distinct cost centers: indexing and inference.33 The indexing phase, which includes chunking and generating vector embeddings, requires compute resources for processing and storage for the resulting vectors.11 Implementing sophisticated structural or semantic chunking (which involves parsing ASTs or calculating semantic distance) requires more initial engineering effort and greater one-time compute for preprocessing compared to the simple character-counting of fixed-size chunking.5  
However, the dominant and ongoing operational expense lies in the inference phase, where LLMs are charged based on the number of input and output tokens passed.4 The use of suboptimal chunking, resulting in the retrieval of large, noisy chunks, directly increases the token count passed to the final LLM, significantly driving up inference cost and latency.3  
There is a clear cost shift: while the initial implementation of advanced, structure-aware chunking may incur a higher setup cost, it results in substantially lower long-term inference costs. The architecture is improved because better chunking yields higher retrieval accuracy, leading to the retrieval of fewer, higher-signal tokens. This reduction in input context length provides substantial savings in ongoing computation and time.34 Strategic resource allocation often involves using smaller, less expensive models for the high-volume retrieval queries and reserving the expensive, high-quality generative LLMs for the final, resource-intensive reasoning step.34 The effectiveness of this cost-saving strategy relies entirely on the precision and quality of the chunks created upstream.

### **B. Optimization through Context Engineering**

The most efficient operational architecture integrates RAG with modern, long-context LLMs.9 RAG is utilized solely to efficiently retrieve highly targeted, relevant chunks, which are then fed into the powerful, large context window LLM for interpretation.35 This architecture is necessary to mitigate the "Lost-in-the-Middle" problem while allowing the LLM to benefit from its full reasoning capacity without being overwhelmed by noisy input.5  
Furthermore, agent-based systems, which utilize external tools for dynamic searching instead of static vector retrieval, offer another layer of cost control. By performing self-managed searches using Bash commands and only loading targeted results into context, these systems minimize the overall token usage, treating context management as a dynamic, iterative process rather than a single, large retrieval event.7

## **VII. Conclusion and Strategic Recommendations for Implementation**

The effectiveness of AI-based code analysis is inextricably linked to the design and implementation of its chunking strategy. The evidence conclusively demonstrates that generic, fixed-size methods are unsuitable for code due to their failure to preserve syntactic and semantic integrity. Success hinges upon adopting structure-aware and hierarchical retrieval systems optimized for the specific downstream task.

### **A. Synthesis of Optimal Strategies for Code RAG**

1. **Mandate Structural Chunking:** The practice of fixed-size chunking must be reserved only for preliminary testing or very uniform documentation. For source code, the chunking method must be structural, aligning with the logical units of the programming language (functions, classes).  
2. **Prioritize AST-Aware Methods:** The implementation of Abstract Syntax Tree (AST)-based chunking, such as cAST, is highly recommended as it empirically demonstrates superior retrieval metrics (Precision and Recall) by ensuring structural integrity.10  
3. **Implement Hierarchical Context (Parent-Child):** To resolve the fundamental trade-off between retrieval precision and contextual completeness, hierarchical retrieval is necessary. Small, precise structural sub-chunks should be used for vector indexing and search, while the larger, complete structural unit (the containing function or class) should be retrieved to ground the LLM's final generation.25

### **B. Decision Framework for Chunking Strategy Selection**

The final choice of chunking size and methodology cannot be monolithic; it must be dynamically selected based on the immediate analytic goal and the required granularity, as summarized in the strategic decision framework below:  
Table III: Decision Framework for Code Chunking Strategy

| Primary Code Analysis Task | Optimal Chunking Strategy | Optimal Size Consideration | Key Performance Metric |
| :---- | :---- | :---- | :---- |
| **Code Generation/Refactoring** | Structural (Function/Class) or Parent-Child | Parent: Full function (500-2000 tokens); Child: Sub-function blocks (100-500 tokens) | Context Relevance, Semantic Coherence |
| **Vulnerability Detection** | Fine-Grained Structural (Micro-Chunking) | Highly localized structural segments (small) | F1 Score, Accuracy |
| **Repository Summarization/Architecture** | File-Level or LLM-Guided Summarization | Large chunks (up to 4096 tokens) with iterative processing | Recall, Cohesion Score |
| **Cross-File Dependency Analysis** | Agentic/Dynamic Search \+ Metadata RAG | Minimal (rely on external tools for traversal) | Architectural Fidelity, Reduced Hallucination |

### **C. Evaluation and Ongoing Optimization**

A formalized system for evaluating the performance of the chunking pipeline is critical for sustained efficacy. This requires setting up testing environments, such as using multiple indices or namespaces, to iteratively test different chunk sizes and strategies against a representative query dataset.5 The quality of the retrieval must be measured against ground-truth references using rigorous metrics like F1 scores and retrieval scores. Furthermore, leveraging LLM-as-a-judge methodologies for evaluating the correctness of generated responses, comparing them against references, and averaging scores across multiple judgment runs ensures the robustness of the optimization process.22

#### **Works cited**

1. What is a context window? \- IBM, accessed October 6, 2025, [https://www.ibm.com/think/topics/context-window](https://www.ibm.com/think/topics/context-window)  
2. Most powerful LLMs (Large Language Models) in 2025 \- Codingscape, accessed October 6, 2025, [https://codingscape.com/blog/most-powerful-llms-large-language-models](https://codingscape.com/blog/most-powerful-llms-large-language-models)  
3. Mastering RAG: Advanced Chunking Techniques for LLM Applications \- Galileo AI, accessed October 6, 2025, [https://galileo.ai/blog/mastering-rag-advanced-chunking-techniques-for-llm-applications](https://galileo.ai/blog/mastering-rag-advanced-chunking-techniques-for-llm-applications)  
4. Why larger LLM context windows are all the rage \- IBM Research, accessed October 6, 2025, [https://research.ibm.com/blog/larger-context-window](https://research.ibm.com/blog/larger-context-window)  
5. Chunking Strategies for LLM Applications \- Pinecone, accessed October 6, 2025, [https://www.pinecone.io/learn/chunking-strategies/](https://www.pinecone.io/learn/chunking-strategies/)  
6. GSM-∞: How Do Your LLMs Behave over Infinitely Increasing Context Length and Reasoning Complexity? \- arXiv, accessed October 6, 2025, [https://arxiv.org/html/2502.05252v1](https://arxiv.org/html/2502.05252v1)  
7. Effective context engineering for AI agents \- Anthropic, accessed October 6, 2025, [https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)  
8. RAG vs. Prompt Stuffing: Overcoming Context Window Limits for Large, Information-Dense Documents \- Spyglass MTG, accessed October 6, 2025, [https://www.spyglassmtg.com/blog/rag-vs.-prompt-stuffing-overcoming-context-window-limits-for-large-information-dense-documents](https://www.spyglassmtg.com/blog/rag-vs.-prompt-stuffing-overcoming-context-window-limits-for-large-information-dense-documents)  
9. Understanding Context window and Retrieval-Augmented Generation (RAG) in Large Language Models | YourGPT, accessed October 6, 2025, [https://yourgpt.ai/blog/general/long-context-window-vs-rag](https://yourgpt.ai/blog/general/long-context-window-vs-rag)  
10. cAST: Enhancing Code Retrieval-Augmented Generation with Structural Chunking via Abstract Syntax Tree \- arXiv, accessed October 6, 2025, [https://arxiv.org/html/2506.15655v1](https://arxiv.org/html/2506.15655v1)  
11. Retrieval Augmented Generation (RAG) for LLMs \- Prompt Engineering Guide, accessed October 6, 2025, [https://www.promptingguide.ai/research/rag](https://www.promptingguide.ai/research/rag)  
12. Five Levels of Chunking Strategies in RAG| Notes from Greg's Video | by Anurag Mishra, accessed October 6, 2025, [https://medium.com/@anuragmishra\_27746/five-levels-of-chunking-strategies-in-rag-notes-from-gregs-video-7b735895694d](https://medium.com/@anuragmishra_27746/five-levels-of-chunking-strategies-in-rag-notes-from-gregs-video-7b735895694d)  
13. Retrieval-augmented generation \- Wikipedia, accessed October 6, 2025, [https://en.wikipedia.org/wiki/Retrieval-augmented\_generation](https://en.wikipedia.org/wiki/Retrieval-augmented_generation)  
14. Does RAG work on large codebases? Or does chunking / embedding ruin an LLM's ability to make sense of an app's code, understand dependencies, etc? : r/LocalLLaMA \- Reddit, accessed October 6, 2025, [https://www.reddit.com/r/LocalLLaMA/comments/1gf2mg5/does\_rag\_work\_on\_large\_codebases\_or\_does\_chunking/](https://www.reddit.com/r/LocalLLaMA/comments/1gf2mg5/does_rag_work_on_large_codebases_or_does_chunking/)  
15. Chunking Strategies to Improve Your RAG Performance \- Weaviate, accessed October 6, 2025, [https://weaviate.io/blog/chunking-strategies-for-rag](https://weaviate.io/blog/chunking-strategies-for-rag)  
16. Which chunker to utilize for code based data \- Intermediate \- Hugging Face Forums, accessed October 6, 2025, [https://discuss.huggingface.co/t/which-chunker-to-utilize-for-code-based-data/145376](https://discuss.huggingface.co/t/which-chunker-to-utilize-for-code-based-data/145376)  
17. Enhancing LLM Code Generation with RAG and AST-Based Chunking | by VXRL \- Medium, accessed October 6, 2025, [https://vxrl.medium.com/enhancing-llm-code-generation-with-rag-and-ast-based-chunking-5b81902ae9fc](https://vxrl.medium.com/enhancing-llm-code-generation-with-rag-and-ast-based-chunking-5b81902ae9fc)  
18. S2 Chunking: A Hybrid Framework for Document Segmentation Through Integrated Spatial and Semantic Analysis \- arXiv, accessed October 6, 2025, [https://arxiv.org/html/2501.05485v1](https://arxiv.org/html/2501.05485v1)  
19. Chunking Analysis: Which is the right chunking approach for your language? \- LanceDB, accessed October 6, 2025, [https://lancedb.com/blog/chunking-analysis-which-is-the-right-chunking-approach-for-your-language/](https://lancedb.com/blog/chunking-analysis-which-is-the-right-chunking-approach-for-your-language/)  
20. Rethinking Chunk Size for Long-Document Retrieval: A Multi-Dataset Analysis \- arXiv, accessed October 6, 2025, [https://arxiv.org/html/2505.21700v2](https://arxiv.org/html/2505.21700v2)  
21. LLM Chunking: How to Improve Retrieval & Accuracy at Scale | Redis, accessed October 6, 2025, [https://redis.io/blog/llm-chunking/](https://redis.io/blog/llm-chunking/)  
22. Finding the Best Chunking Strategy for Accurate AI Responses | NVIDIA Technical Blog, accessed October 6, 2025, [https://developer.nvidia.com/blog/finding-the-best-chunking-strategy-for-accurate-ai-responses/](https://developer.nvidia.com/blog/finding-the-best-chunking-strategy-for-accurate-ai-responses/)  
23. FuncVul: An Effective Function Level Vulnerability Detection Model using LLM and Code Chunk \- arXiv, accessed October 6, 2025, [https://arxiv.org/html/2506.19453v1](https://arxiv.org/html/2506.19453v1)  
24. The Beauty of Parent-Child Chunking. Graph RAG Was Too Slow for Production, So This Parent-Child RAG System was useful \- Reddit, accessed October 6, 2025, [https://www.reddit.com/r/Rag/comments/1mtcvs7/the\_beauty\_of\_parentchild\_chunking\_graph\_rag\_was/](https://www.reddit.com/r/Rag/comments/1mtcvs7/the_beauty_of_parentchild_chunking_graph_rag_was/)  
25. Parent-Child Chunking in LangChain for Advanced RAG | by Seahorse \- Medium, accessed October 6, 2025, [https://medium.com/@seahorse.technologies.sl/parent-child-chunking-in-langchain-for-advanced-rag-e7c37171995a](https://medium.com/@seahorse.technologies.sl/parent-child-chunking-in-langchain-for-advanced-rag-e7c37171995a)  
26. How content chunking works for knowledge bases \- Amazon Bedrock, accessed October 6, 2025, [https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)  
27. LLMs for Generation of Architectural Components: An Exploratory Empirical Study in the Serverless World \- arXiv, accessed October 6, 2025, [https://arxiv.org/html/2502.02539v1](https://arxiv.org/html/2502.02539v1)  
28. Daily Papers \- Hugging Face, accessed October 6, 2025, [https://huggingface.co/papers?q=cross-file%20dependencies](https://huggingface.co/papers?q=cross-file+dependencies)  
29. What Makes Claude Code So Damn Good: A Deep Analysis, accessed October 6, 2025, [https://cc.deeptoai.com/docs/en/advanced/decoding-claude-code-analysis](https://cc.deeptoai.com/docs/en/advanced/decoding-claude-code-analysis)  
30. Can LLMs Replace Humans During Code Chunking? \- arXiv, accessed October 6, 2025, [https://arxiv.org/html/2506.19897v1](https://arxiv.org/html/2506.19897v1)  
31. White-Basilisk: A Hybrid Model for Code Vulnerability Detection \- arXiv, accessed October 6, 2025, [https://arxiv.org/html/2507.08540v2](https://arxiv.org/html/2507.08540v2)  
32. cAST: Enhancing Code Retrieval-Augmented Generation with Structural Chunking via Abstract Syntax Tree \- CMU School of Computer Science, accessed October 6, 2025, [https://www.cs.cmu.edu/\~sherryw/assets/pubs/2025-cast.pdf](https://www.cs.cmu.edu/~sherryw/assets/pubs/2025-cast.pdf)  
33. Decoding RAG Costs: A Practical Guide to Operational Expenses \- Net Solutions, accessed October 6, 2025, [https://www.netsolutions.com/insights/rag-operational-cost-guide/](https://www.netsolutions.com/insights/rag-operational-cost-guide/)  
34. RAG in Practice: Chunking, Context, and Cost \- Reddit, accessed October 6, 2025, [https://www.reddit.com/r/Rag/comments/1nczmmp/rag\_in\_practice\_chunking\_context\_and\_cost/](https://www.reddit.com/r/Rag/comments/1nczmmp/rag_in_practice_chunking_context_and_cost/)  
35. RAG vs Long Context Models \[Discussion\] : r/MachineLearning \- Reddit, accessed October 6, 2025, [https://www.reddit.com/r/MachineLearning/comments/1ax6j73/rag\_vs\_long\_context\_models\_discussion/](https://www.reddit.com/r/MachineLearning/comments/1ax6j73/rag_vs_long_context_models_discussion/)