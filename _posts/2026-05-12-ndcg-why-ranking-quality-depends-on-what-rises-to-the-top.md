---
title: "NDCG: Why Ranking Quality Depends on What Rises to the Top"
date: 2026-05-12
categories: [Machine Learning, Information Retrieval]
tags: [ndcg, information-retrieval, ranking, retrieval-evaluation, recommender-systems]
math: true
---

*If [AUC-ROC vs AUC-PR](/posts/auc-roc-vs-auc-pr-what-changes-when-positives-are-rare/) is about what happens after scores are thresholded, this post is about what happens when the output stays as a ranked list.*

Search engines, recommender systems, and retrieval pipelines for question answering all return the same kind of object: an ordered list.

```text
query → ranked search results
user → ranked recommendations
question → ranked retrieved passages
```

Once the output is a ranking, plain accuracy stops being the right question. A system that puts the best answer at rank 1 is not equivalent to one that buries it at rank 20.

That is what **NDCG** measures.

**NDCG** stands for **Normalized Discounted Cumulative Gain**. In plain language, it asks:

> Compared with the best possible ranking, how much useful relevance did the system place near the top?

That top-heaviness is the whole point. Users inspect early results first. Stanford's information retrieval notes make the assumptions explicit: highly relevant items are more useful than marginally relevant ones, and relevant items are less useful when they appear lower in the list.

## Why this metric exists

Suppose a system returns five documents and human judges assign graded relevance scores:

| Rank | Document | Relevance |
| ---: | --- | ---: |
| 1 | Doc A | 3 |
| 2 | Doc B | 2 |
| 3 | Doc C | 0 |
| 4 | Doc D | 1 |
| 5 | Doc E | 0 |

Interpret the labels as:

```text
3 = highly relevant
2 = relevant
1 = somewhat relevant
0 = irrelevant
```

Now compare two rankings containing the same items:

```text
Good:  3, 2, 1, 0, 0
Worse: 0, 1, 3, 2, 0
```

Both rankings contain the same total relevance. The difference is position. The second one buries the best items lower, so it should score worse.

## How NDCG works

NDCG combines three ideas:

1. **Gain**: more relevant items should contribute more. The simplest choice is linear gain, $\text{gain}_i = \text{rel}_i$. Some implementations use exponential gain, $2^{\text{rel}_i} - 1$, to reward highly relevant items even more strongly.
2. **Discount**: lower-ranked items should count less. The common discount is $1 / \log_2(i + 1)$.
3. **Normalization**: the observed ranking should be compared with the ideal ranking for the same judged item set.

That yields:

$$
\mathrm{DCG}@K = \sum_{i=1}^{K} \frac{\mathrm{rel}_i}{\log_2(i+1)}
$$

and then:

$$
\mathrm{NDCG}@K = \frac{\mathrm{DCG}@K}{\mathrm{IDCG}@K}
$$

where **IDCG** is the DCG of the ideal ordering of the judged items for that query.

The discount pattern is intuitive:

| Rank | Discount |
| ---: | ---: |
| 1 | 1.00 |
| 2 | 0.63 |
| 3 | 0.50 |
| 4 | 0.43 |
| 5 | 0.39 |

So the same relevant document is worth more at rank 1 than at rank 5. Under the usual nonnegative relevance setup, NDCG falls between 0 and 1, where 1 means a perfect ranking.

## A worked example

For a ranking with human-assigned relevance labels $[3, 1, 2]$, the calculation is:

The ideal ordering is $[3, 2, 1]$, so IDCG is computed from that sorted order:

$$
\mathrm{DCG} = 3 + \frac{1}{\log_2(3)} + \frac{2}{2} = 4.631, \quad \mathrm{IDCG} = 3 + \frac{2}{\log_2(3)} + \frac{1}{2} = 4.762
$$

$$
\mathrm{NDCG} = \frac{4.631}{4.762} \approx 0.972
$$

That is high because the best item is already ranked first. The only defect is a small swap between ranks 2 and 3.

## Why teams use NDCG

Accuracy ignores order. It does not care whether the best item is at rank 1 or rank 50.

**Precision@K** is often useful, but it usually treats relevance as binary. NDCG can use binary labels too, but its biggest advantage appears when relevance is graded. In real retrieval systems, that is often the right model: one passage may answer the question directly, another may be partially useful, and a third may only mention the topic in passing.

This is also why **NDCG@K** is common. Most users only inspect the first few results, so evaluation should emphasize the top of the ranking rather than the full list.

## When I would use it

Use NDCG when:

1. the system returns a ranked list
2. top positions matter more than lower positions
3. you have relevance judgments for the ranked items
4. you care about ranking quality, not just retrieval presence

Typical examples include web search, recommender systems, passage retrieval, and RAG pipelines.

## Summary

NDCG is a ranking metric built on a simple idea:

- relevant items are good
- relevant items are better when they appear early
- the observed ranking should be compared with the best possible one

That is why it remains a standard metric in information retrieval. It does not just ask whether good items were retrieved. It asks whether the ranking respected user attention.

For canonical formulations, [Stanford's *Information Retrieval Evaluation* notes](https://web.stanford.edu/class/cs276/handouts/EvaluationNew-handout-1-per.pdf) and Scikit-learn's [`ndcg_score` documentation](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.ndcg_score.html) are both good references.
