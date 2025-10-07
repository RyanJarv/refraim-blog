# Chunking Strategies Deep Dive

Embedded Chunking process described in [https://arxiv.org/html/2506.17798v1\#abstract](https://arxiv.org/html/2506.17798v1#abstract)

Added this here because it’s interesting and goes in more in depth on an embedded chunking implementation covered in [Chunking Strategies Deep Dive](?tab=t.e9eg9kb488m1). 

This phase consists of two main steps:  
---

### **1\. AST-Enhanced Code Segmentation**

The first step is to break down the entire source code into smaller, semantically meaningful blocks3. SAVANT avoids simple techniques like splitting by lines or a fixed number of characters, as these methods can break up important code structures and disrupt the logical flow4.

Instead, it uses  
**Abstract Syntax Trees (ASTs)**, which represent the code's hierarchical structure and semantic relationships5.

Here's how it works:

* **Initial Parsing**: The application's source code is first parsed into an AST6.

* **Balanced Segmentation**: The system must strike a balance.  
  * If the code blocks are too small and fragmented (e.g., treating every single AST node as a block), it becomes difficult to understand the relationships between them and capture the overall logic7777.

  * Conversely, if the blocks are too large (e.g., an entire class file), they may contain a lot of irrelevant code that can confuse the LLM and lead to "hallucinations" or errors8.

* **Heuristic-Based Splitting**: To find the right balance, SAVANT uses a heuristic based on node size. It follows a specific rule for each compilation unit (like a Java file) in the AST9:

  * If a node's size is below a predefined maximum threshold (  
    θ), the entire node is treated as a single block10.

  * If a node's size is larger than the threshold, SAVANT performs a structural segmentation, breaking the unit down into its major sub-nodes. Specifically, it focuses on splitting the code by  
    Import Declaration, Field Declaration, Method Declaration, and Constructor Declaration11. This ensures that the resulting blocks remain logically cohesive.

---

### **2\. Semantic-Preserving Code Embedding**

Once the code is segmented into meaningful blocks, the next step is to convert these blocks into a format that allows for efficient similarity analysis12.

* **Vector Transformation**: Each code block is transformed into a **dense vector embedding**—a numerical representation in a high-dimensional space13131313. This process is designed so that code blocks with similar semantic meanings will be located closer to each other in this vector space14.

* **Using Pre-trained Encoders**: To generate these embeddings, SAVANT uses powerful pre-trained models, such as **OpenAI's V3 embedding models**15.

* **Block-Level Embedding**: SAVANT specifically chose to embed entire blocks rather than individual tokens (words) for two key reasons:  
  1. **Comprehensive Representation**: Block-level embedding captures the overall functionality of a code segment. For example, a traditional for loop that sums elements and a modern Arrays.stream().sum() call are syntactically different but semantically identical. Block-level embedding is more likely to recognize this similarity16161616.

  2. **Standardization**: This approach produces a fixed-length vector for every code block, regardless of its original length. This uniform format enables direct and consistent comparisons between any two code blocks without extra processing17.

### **The Final Output 🧑‍💻**

The output of this entire pre-processing phase is a highly structured **database**. Each entry in this database is a tuple that contains:

* The original code block.  
* Associated metadata, including the filename, line number range, and AST node type.  
* The corresponding vector embedding for that code block18.

This database allows SAVANT to quickly and efficiently search the entire application by semantic meaning, setting the stage for the next phase of vulnerability detection.

