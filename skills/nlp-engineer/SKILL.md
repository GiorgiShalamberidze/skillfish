---
name: nlp-engineer
description: Natural language processing: text classification, tokenization, retrieval, entity extraction, evaluation, and production NLP pipeline design.
---

# NLP Engineer

Design NLP systems from task definition through deployment, not just model selection. This skill emphasizes label quality, text normalization, retrieval and extraction tradeoffs, error-driven evaluation, and production constraints such as latency, drift, privacy, and annotation cost.

## When to Use

- Framing a language task such as classification, extraction, clustering, retrieval, or ranking
- Choosing between rules, classical ML, embeddings, and transformer models
- Building datasets, annotation guidelines, and evaluation sets
- Shipping entity extraction, search, moderation, or document-understanding pipelines
- Debugging poor precision, recall, false positives, or language drift
- Turning notebooks into production NLP systems

## Workflow

1. Define the task in business terms.
   Clarify what must be predicted, extracted, or ranked, and what a costly mistake looks like.
2. Normalize the label space.
   Create annotation guidelines, edge-case rules, and disagreement handling before training anything.
3. Build a strong baseline.
   Start with rules, BM25, TF-IDF, or linear models when appropriate so you know what complexity actually buys you.
4. Select the right model family.
   Use embeddings and retrieval for semantic lookup, sequence models for token-level extraction, and classifier heads for fixed taxonomies.
5. Evaluate by error bucket.
   Slice by class, language, document length, source, and ambiguity instead of hiding behind one aggregate score.
6. Plan production behavior.
   Define latency budget, fallback behavior, monitoring, retraining cadence, and privacy constraints before launch.

## Model Guide

| Approach | Best For | Weakness |
|---|---|---|
| Rules / regex | High-precision patterns and compliance checks | Brittle coverage and maintenance cost |
| TF-IDF / linear models | Fast baselines on clean labeled text | Weak semantic generalization |
| Embeddings + retrieval | Semantic search, clustering, duplicate detection | Limited structured reasoning on top of matches |
| Transformer classifiers | Rich classification and extraction tasks | Higher cost, serving complexity, and drift sensitivity |

## Evaluation Rules

- Match metrics to the task: precision/recall/F1 for classification and extraction, MRR or nDCG for ranking, exact match only when justified.
- Keep a frozen evaluation set plus a "recent production" shadow set.
- Review false positives and false negatives weekly after launch.
- Track class imbalance explicitly; accuracy alone is usually misleading.
- Measure annotation disagreement because low label quality caps model quality.

## Production Checklist

- Text normalization is versioned and testable
- Training, eval, and inference schemas are consistent
- Sensitive text handling is defined before data collection
- Thresholds are tuned for business risk, not generic defaults
- Monitoring covers input drift, label drift, and latency
- Human review path exists for ambiguous or high-impact outputs

## Output Format

When using this skill, produce:

1. **Task definition** - goal, labels, failure cost, and success metrics
2. **Pipeline choice** - baseline, model family, retrieval/extraction steps, and rationale
3. **Data plan** - annotation rules, dataset slices, and evaluation strategy
4. **Deployment plan** - serving path, thresholds, fallback, and monitoring
5. **Improvement loop** - error buckets, retraining cadence, and human-review triggers
