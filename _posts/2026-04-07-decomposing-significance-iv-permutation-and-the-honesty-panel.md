---
title: "Decomposing Significance IV: Permutation Testing and the Honesty Panel"
date: 2026-04-07
categories: [Statistics]
tags: [permutation-testing, diagnostics, count-data, reproducibility, rank-stability]
math: true
---

*This is Part IV of the **Decomposing Significance** series: [Part I](/posts/decomposing-significance-i-anatomy-of-a-test-statistic/) | [Part II](/posts/decomposing-significance-ii-borrowing-strength/) | [Part III](/posts/decomposing-significance-iii-effect-size-vs-variance/) | Part IV (this article)*

---

## The ground truth test

Parts I–III built the conceptual framework and a set of visual diagnostics. This final part introduces one of the most direct checks — permutation — and assembles everything into a single diagnostic panel that answers one question:

> "Should I trust these significance calls, and if so, why?"

## Permutation testing: removing the signal

The idea is devastatingly simple.

If the labels (group A vs group B) carry real information about the features, then scrambling the labels should destroy the significance. If it does not — if you still get hundreds of significant features after random label permutation — your pipeline is producing artifacts.

### The procedure

1. Take your original count matrix: $$n$$ samples × $$p$$ features
2. Randomly permute the condition labels across samples
3. Run the full differential testing pipeline (same model, same shrinkage, same FDR threshold)
4. Count the number of significant features
5. Repeat 100–1,000 times

### Interpreting the result

The permutation distribution of significant feature counts gives you the **null expectation**: how many features would be called significant if the labels were meaningless.

| Observed vs permutation | Interpretation |
| :------------------------ | :--------------- |
| Observed $$\gg$$ permutation | Strong evidence that the labels carry information beyond the null permutations |
| Observed $$\approx$$ permutation | No detectable signal — observed results are consistent with noise |
| Observed $$>$$ permutation, but modestly | Weak signal, likely inflated by variance artifacts |

### The critical insight

Under a well-calibrated complete null, FDR-controlled pipelines often produce few or no discoveries on permuted data. The exact number depends on the testing procedure, dependence structure, and the shape of the p-value distribution under permutation, so the empirical permutation distribution itself is the benchmark.

If permutation repeatedly produces a substantial number of significant features, something is systematically wrong. Common causes:

- **Shrinkage miscalibration**: the empirical Bayes prior is too aggressive, compressing variance estimates so far that even random label assignments produce inflated test statistics
- **Structured noise**: the samples have batch effects, technical confounders, or correlation structure that the permutation does not respect. Permutation assumes exchangeability under the null, and if samples are not exchangeable (e.g., because of batch effects), permuted labels can be anti-correlated with the confounders, producing spurious significance
- **Model misspecification**: the negative binomial or linear model does not fit the data, and the residual structure inflates test statistics

Each of these is diagnosable. The permutation test tells you *that* something is wrong; the other diagnostics from Part III tell you *what*.

### Restricted permutation for structured designs

In designs with blocking factors, batches, or paired samples, simple permutation is incorrect — it breaks the correlation structure that the model accounts for.

The fix is restricted permutation: permute labels only within blocks. If samples are paired (e.g., before/after within the same unit), permute the labels within each pair. If there are batches, permute within each batch. This respects the exchangeability structure that the null hypothesis actually implies.

Getting this right matters. Unrestricted permutation in a blocked design can inflate the permutation null, making your observed results look less extreme than they really are. Restricted permutation gives a calibrated null.

## Rank stability: do your top features survive perturbation?

Significance calls are binary: a feature is either significant or not. But the ranking of features by test statistic or p-value carries more information. If the ranking is stable under perturbation, the top features are robust. If the ranking shuffles easily, even the strongest-looking results are fragile.

### Replicate removal

1. Remove one replicate from each condition
2. Re-run the differential testing pipeline
3. Compare the top-k feature list to the original

Repeat for each replicate. Compute the overlap between the perturbed top-k and the original top-k across all leave-one-out runs.

### What stability looks like

**Effect-driven significance**: the top features are driven by large effects that are visible across replicates. Removing one replicate changes the standard errors but not the effect estimates much. The top-k list is largely preserved. Overlap is 70-90%.

**Variance-driven significance**: the top features are driven by small effects that happen to coincide with low variance estimates. Removing one replicate can drastically change the variance estimate for a feature, promoting or demoting it. The top-k list shuffles substantially. Overlap is 30-50%.

### Downsampling

An alternative to leave-one-out: randomly subsample (say, 80% of samples) and re-run. Repeat 50 times. Compute pairwise overlap of top-k lists.

If the median pairwise overlap is below 60%, your significance ranking is unstable — it is driven by specific samples rather than by consistent effects.

### Quantifying rank stability

A useful metric is the **rank-biased overlap (RBO)**, which compares two ranked lists with a top-weighting parameter. For two ranked lists $$L_1$$ and $$L_2$$:

$$
\text{RBO}(L_1, L_2; p) = (1 - p) \sum_{d=1}^{\infty} p^{d-1} \cdot \frac{|L_1[:d] \cap L_2[:d]|}{d}
$$

where $$p \in (0, 1)$$ controls how much weight the top of the list receives. Higher $$p$$ weights deeper ranks; $$p = 0.9$$ is a common default.

RBO ranges from 0 (completely different rankings) to 1 (identical rankings). An RBO below 0.5 across replicate-removal perturbations is a strong signal that the ranking is variance-driven.

## The honesty panel

Here is the single-page diagnostic panel that combines everything from this series. Generate all six components and lay them out together.

### Panel layout

```text
┌─────────────────────────┬─────────────────────────┐
│  (A) |β| distribution   │  (B) Volcano plot        │
│  All vs significant      │  p vs |β| coupling       │
│                          │                          │
├─────────────────────────┼─────────────────────────┤
│  (C) Mean-variance       │  (D) SNR decomposition   │
│  By condition            │  Δμ vs σ component shift  │
│                          │                          │
├─────────────────────────┼─────────────────────────┤
│  (E) Permutation null    │  (F) Rank stability       │
│  Observed vs permuted    │  Leave-one-out overlap    │
│  N_sig distribution      │  across perturbations     │
└─────────────────────────┴─────────────────────────┘
```

### How to read the panel

**Panel (A): Effect size distribution.** ECDF or histogram of $$|\hat{\beta}_j|$$ for all features (gray) and significant features (colored). If the colored curve barely shifts right, effects are small. Variance is driving significance.

**Panel (B): Volcano plot.** $$-\log_{10}(p)$$ vs $$|\hat{\beta}_j|$$. Diagonal = effect-driven. Vertical band at small $$|\hat{\beta}|$$ = variance-driven. Look at the left edge: if the most significant features cluster near $$|\hat{\beta}| = 0$$, that is a strong warning sign.

**Panel (C): Mean-variance plot.** Separate trends per condition. If one trend sits below the other or has a different shape, variance structure differs between conditions. Overlay the fitted shrinkage trend (dashed line) to see how much the model compresses.

**Panel (D): SNR decomposition.** Two density plots: the distribution of $$|\hat{\beta}_j|$$ and the distribution of $$\hat{\sigma}_j$$, comparing across conditions. Which one shifted? If the model-based noise term $$\hat{\sigma}$$ shifted and $$|\hat{\beta}|$$ did not, the significance difference is variance-driven.

**Panel (E): Permutation null.** Histogram of $$N_{\text{sig}}$$ across permutations, with a vertical line at the observed value. If the observed value is far into the tail (say, $$> 99\%$$ of permutations), the evidence is much stronger than the null permutation baseline. If it is in the body of the distribution, the signal is not distinguishable from label noise.

**Panel (F): Rank stability.** Bar chart or box plot of top-k overlap across leave-one-out or downsampling perturbations. Overlap above 70% = stable. Overlap below 50% = fragile.

### The decision framework

After generating the panel, classify the result:

| Panel A | Panel B | Panel C | Panel E | Panel F | Verdict |
| :-------- | :-------- | :-------- | :-------- | :-------- | :-------- |
| Shifted | Diagonal | Stable | Far tail | High overlap | **Effect-driven** — trust the results |
| Static | Vertical | Shifted | Moderate | Low overlap | **Variance-driven** — decompose before interpreting |
| Mixed | Mixed | Mixed | Tail | Moderate | **Mixed** — report effect size estimates, not just counts |

## What to do when significance is variance-driven

Discovering that your significance counts are variance-driven is not a failure. It is information. The appropriate response depends on the context.

### Option 1: Report effect sizes, not counts

Instead of "2,000 features are significant," report "the median absolute log fold change among significant features is 0.3, and the interquartile range is [0.15, 0.6]." This separates the statistical claim (we detected effects) from the magnitude claim (the effects are this large).

If the median $$|\hat{\beta}|$$ is 0.3, the effects are real but small. Whether that matters depends on the application. In some domains, a 23% shift ($$2^{0.3}$$) is enormous. In others, it is measurement noise.

### Option 2: Use effect size thresholds

Apply a minimum $$|\hat{\beta}|$$ cutoff alongside the FDR threshold. This eliminates the variance-driven significant features that have negligible effects. The resulting list is smaller but more interpretable.

The choice of cutoff requires domain knowledge. What constitutes a "meaningful" effect depends on the application, the measurement precision, and the downstream use of the results.

### Option 3: Compare conditions on effect distributions, not counts

If you are comparing two conditions, compare the full distributions of $$\hat{\beta}_j$$ rather than the counts of significant features. Two conditions can have identical effect distributions but different significance counts (because of different variance structures). The effect distribution comparison strips out the variance artifact.

Metrics for this comparison: Kolmogorov-Smirnov test on the $$\hat{\beta}_j$$ distributions, comparison of quantiles, or simply overlaid density plots.

### Option 4: Investigate the variance difference

If two conditions have different variance structures, that itself is interesting and worth investigating. Why is one condition quieter? Is it technical (different protocols, different noise levels, different sample quality) or is it a real consequence of the condition (the intervention stabilizes the system)?

This reframes the question: instead of "which condition produces more significant features?" ask "why does one condition have lower variance?" That question may be more informative than the original analysis.

## The one sentence

If there is a single takeaway from this series, it is this:

> "The number of significant features in a high-dimensional differential test is a joint function of effect magnitude and variance estimation precision. Reporting the count without decomposing its components conflates statistical sensitivity with effect strength."

This is a factual statement about how the methods work. It applies to every pipeline that uses test-statistic-based significance in count data. It is not a criticism of any particular method — it is a description of what the methods can and cannot tell you.

The diagnostics in this series are the tools to make that decomposition explicit. Use them routinely, and you will never mistake a quiet experiment for a strong one.

---

*Previous: [Part III — Effect Size vs Variance: What Actually Drove Your Results?](/posts/decomposing-significance-iii-effect-size-vs-variance/)*

*Return to: [Part I — The Anatomy of a Test Statistic in Count Data](/posts/decomposing-significance-i-anatomy-of-a-test-statistic/)*
