---
title: "AUC-ROC vs AUC-PR: What Changes When the Positives Are Rare?"
date: 2026-04-19
categories: [Machine Learning]
tags: [auc-roc, auc-pr, model-evaluation, class-imbalance, computational-biology]
math: true
---

## The metric can look strong while the model is operationally weak

Suppose you are building a classifier for a rare event: a fraudulent transaction, a readmission risk, or a screening hit in a large assay where only a small fraction of candidates are truly positive.

The first metric many teams report is AUC-ROC. If it is 0.92, the model sounds excellent. But under heavy class imbalance, that number can hide the question the user actually cares about:

> "If I act on the cases this model flags as positive, how many of those flags will be correct?"

That is why AUC-ROC and AUC-PR are not interchangeable. They summarize different tradeoffs, and they behave very differently once positives become rare.

## The two curves are asking different questions

### ROC: how well do we separate positives from negatives?

The ROC curve plots:

- **True positive rate**: $\text{TPR} = \frac{TP}{TP + FN}$
- **False positive rate**: $\text{FPR} = \frac{FP}{FP + TN}$

As you sweep the classification threshold, you trace a curve in $\text{FPR}$-$\text{TPR}$ space. The AUC-ROC is the area under that curve.

Its interpretation is ranking-based: if you draw one random positive and one random negative, ROC AUC is the probability that the model assigns the positive a higher score than the negative.

That is a useful quantity. But it is not the same as asking whether flagged positives are trustworthy.

### PR: when we predict positive, how often are we right?

The precision-recall curve plots:

- **Recall**: $\text{Recall} = \frac{TP}{TP + FN}$
- **Precision**: $\text{Precision} = \frac{TP}{TP + FP}$

Again, the threshold moves and the curve changes. But now the vertical axis is precision rather than false positive rate.

That changes the meaning completely. PR focuses on the part of the confusion matrix that matters when positives are rare: the competition between true positives and false positives among the predicted positives.

## Why imbalance changes the story

The reason is in the denominators.

ROC uses:

$$
\text{FPR} = \frac{FP}{FP + TN}
$$

If the negative class is huge, the denominator $FP + TN$ is huge too. That means you can accumulate a fairly large number of false positives while keeping FPR numerically small.

PR uses:

$$
\text{Precision} = \frac{TP}{TP + FP}
$$

Here, every false positive directly competes with true positives in the denominator. When positives are rare, precision can collapse quickly even while ROC still looks respectable.

This is the central reason PR is often more informative for rare-event problems.

## A concrete example

Assume 10,000 samples:

- 100 positives
- 9,900 negatives

At one threshold, the model produces:

- $TP = 80$
- $FN = 20$
- $FP = 198$
- $TN = 9{,}702$

Now compute the relevant rates.

### ROC view

$$
\text{TPR} = \frac{80}{100} = 0.80
$$

$$
\text{FPR} = \frac{198}{9900} = 0.02
$$

That ROC operating point, $\text{FPR} = 0.02$ and $\text{TPR} = 0.80$, looks strong. Many readers would call it a high-quality classifier.

### PR view

$$
\text{Precision} = \frac{80}{80 + 198} \approx 0.288
$$

$$
\text{Recall} = 0.80
$$

Now the same operating point looks different: fewer than 29% of flagged positives are real.

That may be excellent or unacceptable depending on the workflow. At 1% prevalence, 28.8% precision is far above random. But it also means most alerts are still false positives, which can be operationally expensive if each flagged case triggers review, lab follow-up, or intervention.

Nothing changed about the model. Only the metric lens changed.

That is the practical issue. Under class imbalance, ROC can make the model look operationally cleaner than it is.

## The random baseline behaves differently too

This is another place where people get misled.

### ROC baseline

For a random classifier, the ROC AUC baseline is:

$$
\text{AUC-ROC}_{\text{random}} = 0.5
$$

More precisely, if the model scores are independent of the true labels, the expected ROC curve is the diagonal line:

$$
\text{TPR} = \text{FPR}
$$

and the area under that diagonal is 0.5. This is true regardless of class prevalence.

### PR baseline

For a random classifier, the PR baseline is the positive prevalence:

$$
\text{Precision}_{\text{baseline}} = \frac{P}{P + N}
$$

If positives are 1% of the dataset, the baseline precision is 0.01.

The intuition is different from ROC. If the ranking is random, then as you move down the ranked list, the expected fraction of positives you collect stays equal to the dataset prevalence. So the no-skill PR level sits at the prevalence, and the corresponding average precision baseline is:

$$
\text{AP}_{\text{random}} = \frac{P}{P + N}
$$

In a 1%-positive problem, a random model has ROC AUC 0.5, and its average precision is about 0.01.

That means PR is inherently prevalence-sensitive. This is a feature, not a bug: it reflects the actual difficulty of finding needles in a haystack.

But it also means PR AUC values should not be compared across datasets without noting prevalence. A PR AUC of 0.20 may be weak in a balanced dataset and very strong in a 1%-positive dataset.

This point has been emphasized repeatedly in the literature on classifier evaluation under imbalance, including the relationship between ROC and PR geometry described by Davis and Goadrich (2006) and the practical recommendation to prefer PR plots for rare positives stressed by Saito and Rehmsmeier (2015).

## When ROC is still useful

This is not an anti-ROC article. ROC remains useful when:

- the classes are not severely imbalanced
- the goal is ranking quality across thresholds rather than predicted-positive purity
- false positives and false negatives are both important, but neither class is vanishingly rare
- you want a prevalence-insensitive measure of separability

ROC is often a good diagnostic for whether the score distribution contains signal at all.

If the ROC AUC is near 0.5, the model is not separating classes. That is a basic failure.

## When PR is the better default

PR becomes more informative when:

- positives are rare
- the operational question is "how trustworthy are the positive calls?"
- follow-up cost is high
- the workflow acts only on the top-scoring or thresholded positives

This is common in:

- fraud detection
- adverse event prediction
- rare disease screening
- hit discovery in large screening campaigns
- retrieval-style ranking problems where only a small top slice will be reviewed

In these settings, the user rarely cares about negative-class ranking in the abstract. They care about what happens when the system flags a case.

## AUC-PR is not the whole answer either

PR is often the better summary under imbalance, but it still does not solve everything.

### Threshold choice still matters

Two models can have similar AUC-PR and very different precision at the specific recall you need. If your workflow requires at least 70% precision, the metric you really care about may be:

- recall at fixed precision
- precision at fixed recall
- precision in the top-k predictions

The summary area is useful, but operational decisions happen at thresholds.

### Calibration is separate

A model can have a good ROC AUC or PR AUC and still produce poorly calibrated probabilities. Good ranking does not imply well-calibrated risk estimates.

If users act on predicted probabilities rather than just ranks, calibration needs to be checked separately.

### "AUC-PR" is not always one unique number

Some libraries report trapezoidal PR AUC. Others emphasize **average precision (AP)**, which uses a different interpolation convention and is often preferred in information retrieval and imbalanced classification.

Those numbers are related but not identical. A report should state which one was used.

## The practical reporting standard

For rare-event classification, a stronger evaluation bundle is:

1. ROC AUC for broad separability
2. PR AUC or average precision for rare-positive performance
3. Prevalence of the positive class
4. Precision and recall at the deployment threshold
5. Calibration check if probabilities will be consumed directly
6. Uncertainty intervals when the positive count is small, for example via stratified bootstrap

That package tells a much more honest story than ROC AUC alone.

## The decision rule

If you are working on an imbalanced problem, PR-family reporting is usually closer to the operational truth than ROC alone.

The reason is simple: in rare-event settings, the cost of false positives is not diluted by the huge negative class in the way ROC allows.

ROC asks whether positives outrank negatives.

PR asks whether predicted positives are worth believing.

When positives are rare, that second question is often closer to the real operating problem. But if the workflow acts on a fixed review budget or only the top-ranked cases, pair PR-style summaries with the thresholded or top-k metric the workflow actually uses.

References: [Davis and Goadrich, 2006](https://dl.acm.org/doi/10.1145/1143844.1143874); [Saito and Rehmsmeier, 2015](https://doi.org/10.1371/journal.pone.0118432).
