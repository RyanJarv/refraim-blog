# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repository contains source files and analysis for a technical blog post evaluating code chunking strategies for AI-powered vulnerability scanning. The blog analyzes how different methods of splitting code affect the accuracy, speed, and cost of LLM-based security scanning using the **fraim** vulnerability scanner.

## Key Commands

Since this is primarily a documentation and analysis repository, there are no build or test commands. The blog post in `blog.md` is the primary deliverable.

## Architecture Overview

### Document Structure

The repository follows a hierarchical documentation structure:

- **`blog.md`**: The primary blog post draft
- **`Overview.md`**: Canonical outline and structure for the blog (source of truth for organization)
- **`AGENT.md`**: AI agent instructions for working on the blog
- **`GEMINI.md`**: High-level directory overview
- **`STYLE.md`**: Comprehensive style guide (mandatory for all writing)
- **`TODO.md`**: High-level goals and direction (not a task list)

### Supporting Materials

- **`info/`**: Analysis, process notes, and supporting research
  - `analysis.md`: Canonical benchmark conclusions (source of truth for findings)
  - `process.md`: Testing process and technical reference
  - `slack.md`: Team conversations and decisions
  - `results_explanations.md`: Metrics and methodology explanations
- **`results/`**: Benchmark outputs
  - `results.md`: Summary tables with metrics
  - `*.html`: Per-dataset visualizations
- **`docs/`**: Deep-dive research on chunking strategies
- **`examples/`**: Case studies and reference implementations
- **`datasets.md`**: Dataset descriptions and characteristics
- **`code/`**: Symlinks to inline code references

## Writing Workflow

### Content Hierarchy

1. **`Overview.md`** defines section intent, hypotheses, and structure
2. **`info/analysis.md`** contains final benchmark conclusions
3. **`results/results.md`** provides current quantitative data
4. **`STYLE.md`** governs all writing decisions
5. **`blog.md`** implements the above in final form

### Key Principles

- **Never modify hypotheses in Overview.md** - they represent original expectations
- **Always ground findings in `results/results.md`** and mirror conclusions from `info/analysis.md`
- **Follow STYLE.md strictly** for voice, tone, structure, and formatting
- **Use Overview.md to drive blog.md structure** - treat "Description" as required coverage
- **Maintain separation between strategy description (in "Exploring Chunking Strategies") and results (in "Results and Analysis")**

### Content Organization Rules

#### Exploring Chunking Strategies Section
- Describe how each chunker works (project, file, fixed-size, syntactic, packed, packed_fixed)
- State the original hypothesis for each strategy
- Provide light qualitative context only
- **Do NOT include hard metrics or detailed results here**

#### Results and Analysis Section
- Present quantitative findings with specific numbers from `results/results.md`
- Compare methods dataset-by-dataset and cross-method
- Include timing data and performance trade-offs
- Cite metrics: Precision, Recall, F1, Processing Time, Skipped Lines
- Keep conclusions synchronized with `info/analysis.md`

#### Methodology Section
- Document benchmark scope (scan only baseline-referenced files; triage disabled)
- Explain comparator mechanics and matching criteria
- Note limitations explicitly

## Chunking Strategies Covered

The blog analyzes these approaches:

1. **project**: Concatenate whole project up to token window
2. **file**: One file per chunk, no splitting
3. **original**: Line-based splits (sizes in lines, not tokens)
4. **fixed_token**: Token-based splits with fixed boundaries
5. **syntactic**: Language-aware splitting at function/class boundaries (LangChain-based, not AST)
6. **packed**: Pack whole files to fill token budget efficiently; split oversized files with syntactic
7. **packed_fixed**: Same as packed but split oversized files with fixed_token

## Benchmark Context

### Datasets

- **generated/***: 12 single-vulnerability projects for sanity checks (not decision drivers)
- **repos/juice-shop**: 75 findings across 32 files; hardest due to scale and cross-file dependencies
- **repos/validation-benchmarks**: 30 findings across 20 files; longer spans with unrelated test cases
- **repos/verademo**: 65 findings across 19 files; heavy same-file clustering

### Key Findings (from info/analysis.md)

- Mid-sized packed (≈5.5-7k tokens) yields best average F1
- Larger packed sizes raise precision but reduce recall
- Project strategy compares closely with large packed sizes and can be faster
- Syntactic tracks fixed-size and sometimes underperforms slightly
- Overlap not reliably enforced by LangChain splitters

### Units Policy

- All chunkers use **token** budgets except `original`, which uses **lines**
- Keep labels explicit in text and figures
- "Skipped Lines / 100k" normalizes unscanned lines per 100,000 lines for cross-repo comparison

## Metrics Definitions

- **True Positives (TP)**: Correct vulnerability detections
- **False Positives (FP)**: Incorrect detections
- **False Negatives (FN)**: Missed vulnerabilities
- **Precision**: TP / (TP + FP) - signal-to-noise ratio
- **Recall**: TP / (TP + FN) - percentage of real vulnerabilities found
- **F1-Score**: Harmonic mean of precision and recall
- **Processing Time**: Average total processing time per configuration
- **Skipped Lines / 100k**: Unscanned lines normalized to per-100k scale

## Important Constraints

### What NOT to Do

- Do not copy content from STYLE.md into other files (reference it instead)
- Do not modify hypotheses in Overview.md (they are frozen source material)
- Do not use Generated datasets to drive conclusions (they are sanity checks only)
- Do not merge "When to Use" into "Hypothesis" in Overview.md
- Do not include hard metrics in the "Exploring Chunking Strategies" section
- Do not diverge from Overview.md structure without explicit approval

### What TO Do

- Update `info/analysis.md` first when understanding of results changes
- Keep blog.md synchronized with analysis.md conclusions
- Verify all dataset counts and metrics against current `results/results.md`
- Use layered explanation protocol from STYLE.md (nested doll approach)
- Include reproducibility instructions (CLI commands, API keys, dataset paths)
- Maintain concise, defensive storytelling focused on security engineering readers

## Comparison Mechanics

The benchmark comparison system:
- Matches on vulnerability `type` + at least one overlapping file
- Uses semantic similarity (SentenceTransformers + optional CodeBERT) over messages/explanations
- Requires adaptive similarity threshold (default 0.85)
- Each expected result matches at most once
- False negatives inside unscanned spans are excluded from recall calculation
- Precision reflects noise within vulnerable files (not full-repo precision)

## Future Considerations

From TODO.md:
- Units policy decision: annotate CLI test names with units or clarify in docs only
- Revisit largest chunk sizes after skipped-lines handling fix
- Optional: Add "Key Takeaways" bullet list to Conclusion

## Related Tools

The blog references but does not implement:
- **fraim**: The vulnerability scanner being benchmarked
- **refraim**: Tooling that runs fraim against datasets and compares results
- **simonw/files-to-prompt** and **simonw/ttok**: Token counting utilities

## Style Requirements

All writing must follow STYLE.md requirements:
- Active voice and imperative mood
- Layered narrative aligned with investigation chronology
- Redacted placeholders for sensitive data (UPPER_SNAKE_CASE)
- Fenced code blocks with language tags, no prompts
- Warning callouts before risky operations
- One H1 per file with logical H2/H3 hierarchy
- Alt text for all visuals
- Cross-links to related content without duplication
