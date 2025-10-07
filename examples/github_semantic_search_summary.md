# Towards Natural Language Semantic Code Search (Summary)

This is a summary of the GitHub blog post "Towards Natural Language Semantic Code Search".

**Motivation:**
Current code search on GitHub is limited to keyword search, requiring users to know syntax or anticipate keywords in comments. Semantic search aims to overcome this by understanding the meaning behind a query, even if keywords don't directly match the code. This would expedite onboarding and improve code discoverability.

**Introduction:**
GitHub's machine learning research focuses on learning representations of entities like repos, code, and issues. They are making progress by learning code representations that share a common vector space with natural language. This allows (text, code) pairs describing the same concept to be close neighbors in the vector space.

**Approach (Four Parts):**

1.  **Learning Representations of Code:**
    *   A sequence-to-sequence model is trained to summarize code (e.g., using `(code, docstring)` pairs for Python).
    *   The encoder from this model is then used as a general-purpose feature extractor for code.
    *   This model can be evaluated using the BLEU score (achieved 13.5 on a Python holdout set).

2.  **Learning Representations of Text Phrases:**
    *   Initially experimented with Universal Sentence Encoder, but found it advantageous to train domain-specific embeddings for software development vocabulary.
    *   A neural language model was trained using the `fast.ai` library, leveraging architectures like AWD LSTMs and cyclical learning rates.
    *   Embeddings are sanity-checked by manually examining similarity between phrases.

3.  **Mapping Code Representations to the Same Vector-Space as Text:**
    *   The code encoder (from part 1) is fine-tuned to map code representations into the vector space of text.
    *   The target variable for this fine-tuning is the vectorized version of docstrings (from part 2).
    *   Multi-dimensional regression with cosine proximity loss is used to align the hidden states.

4.  **Creating a Semantic Search System:**
    *   Once code can be vectorized into the same space as text, a semantic search mechanism can be created by storing vectorized code and performing nearest neighbor lookups for vectorized search queries.
    *   Ongoing research includes augmenting keyword search with semantic results and incorporating context/relevance.

**Summary:**
The process involves learning code representations, learning text phrase representations, mapping code representations to the text vector space, and then building a semantic search system based on nearest neighbor lookups.

**Open Source Examples:**
An end-to-end tutorial with code and data is available to reproduce the results, and it's also used as a tutorial for the Kubeflow project.

**Limitations and Intended Use Cases:**
*   Semantic code search is expected to be most useful for targeted searches within specific entities (repos, organizations, users), rather than general "how-to" queries.
*   The efficacy is limited by training data (e.g., `(code, docstring)` pairs), meaning queries resembling docstrings have the highest chance of success.
*   The live demonstration is a minimal example and can be challenged to reveal its limitations.
*   Future work includes extending search to different natural languages and programming languages.