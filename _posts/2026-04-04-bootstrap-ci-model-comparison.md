---
title: "Bootstrap CI for Model Comparison: A Practical Guide"
date: 2026-04-04
categories: [Statistics]
tags: [bootstrap, confidence-intervals, model-comparison, computational-biology, python]
math: true
---

## Why this matters

In model comparison reports, you often see statements like "Bootstrap 95% CI on delta excludes zero." This sentence is statistically important, but it can sound abstract. This guide explains exactly what it means, why it matters, and how to verify it in practice.

## The core idea in one sentence

When comparing two models, we estimate the performance difference many times by resampling the data. If the 95% confidence interval for that difference does not cross zero, we have evidence that the difference is real — not just random noise.

## Step 1: Define what you compare

Assume you compare Model A vs Model B. For each evaluation unit (e.g., a gene), compute a metric (often Pearson *r*) for both models, then compute a paired difference:

$$\delta_{\text{gene}} = \text{metric}_{A,\text{gene}} - \text{metric}_{B,\text{gene}}$$

If δ is positive, Model A did better on that unit.

Using paired differences is critical because both models are evaluated on the same units.

## Step 2: Why bootstrap instead of only a t-test

In biology-facing ML work, we often have a modest number of evaluation units (say, 15 genes), and metric distributions may not be perfectly normal. Bootstrap helps because it:

- Is **non-parametric** — fewer distributional assumptions
- Works well with **small-to-moderate sample sizes**
- Directly estimates **uncertainty of the observed mean difference**

## Step 3: The bootstrap procedure (paired)

Given *N* paired deltas (one per evaluation unit):

1. Sample *N* deltas **with replacement**
2. Compute the mean δ of that resample
3. Repeat many times (e.g., 10,000)
4. Take the **2.5th** and **97.5th** percentiles of bootstrap mean deltas

Those two percentiles form the **95% bootstrap confidence interval**.

## Step 4: Interpreting "excludes zero"

Let CI = [lower, upper].

| Condition | Interpretation |
| --------- | -------------- |
| lower > 0 **and** upper > 0 | Model A is significantly better |
| lower < 0 **and** upper < 0 | Model B is significantly better |
| lower ≤ 0 ≤ upper | No clear winner at 95% confidence |

**Example:** CI = [+0.004, +0.088]

Both bounds are positive, so zero is not inside the interval. This supports a real positive advantage for Model A on the chosen metric.

## Intuition

Think of bootstrap as asking:

> "If I reran this study many times with similar data variability, what range of average model differences would I usually see?"

If that range is entirely above zero, "no difference" is not consistent with the data.

## Practical verification checklist

Before claiming significance, verify all of these:

1. **Pairing is correct** — Compare A and B on identical evaluation units (same genes/folds/samples)
2. **Delta sign is documented** — Clearly state δ = A − B (not B − A)
3. **Resampling is paired** — Resample units (genes) and keep each pair together
4. **Enough bootstrap iterations** — At least 5,000; 10,000 is common
5. **CI percentile method is stated** — Usually percentile CI: [2.5th, 97.5th]
6. **CI reported with signs** — Example: [+0.004, +0.088], not [0.004, 0.088]
7. **Biological relevance is discussed** — Statistical significance does not automatically mean practical utility

## Common mistakes

- **Mixing unpaired and paired analysis** — if models are evaluated on the same units, the analysis must be paired
- **Changing metric direction** (higher-is-better vs lower-is-better) without adjusting delta sign
- **Bootstrapping at the wrong level** — e.g., resampling individual siRNAs when the claim is gene-level generalization
- **Reporting only p-values** without effect size and CI

## Minimal Python example

```python
import numpy as np

# Per-gene Pearson r from two models (same genes, paired)
r_a = np.array([0.61, 0.52, 0.40, 0.55, 0.48,
                0.58, 0.44, 0.50, 0.47, 0.62,
                0.43, 0.54, 0.46, 0.57, 0.49])
r_b = np.array([0.56, 0.49, 0.39, 0.51, 0.45,
                0.54, 0.42, 0.47, 0.44, 0.58,
                0.41, 0.50, 0.43, 0.52, 0.46])

delta = r_a - r_b  # define direction explicitly
obs_mean_delta = delta.mean()

rng = np.random.default_rng(42)
n_boot = 10_000
boot_means = np.empty(n_boot)

n = len(delta)
for i in range(n_boot):
    idx = rng.integers(0, n, size=n)
    boot_means[i] = delta[idx].mean()

ci_low, ci_high = np.percentile(boot_means, [2.5, 97.5])

print(f"Observed mean delta (A-B): {obs_mean_delta:.4f}")
print(f"95% bootstrap CI: [{ci_low:+.4f}, {ci_high:+.4f}]")
print("Excludes zero?", not (ci_low <= 0 <= ci_high))
```

Running this produces a CI entirely above zero, confirming a statistically supported advantage for Model A.

## Reporting template

Use this style in papers, decks, or model cards:

> "We compared models using paired per-gene Pearson *r* differences (A−B). A 10,000-iteration paired bootstrap produced a 95% CI of [*L*, *U*] for the mean delta. Because the interval excluded zero, Model A showed a statistically supported advantage over Model B."

Optional add-on: "Observed mean δ = *X* (*Y*% relative improvement)."

## Final takeaway

"Bootstrap CI excludes zero" is a rigorous way to say the model advantage is unlikely to be due to random sampling variation — especially valuable in small-to-moderate biological evaluation sets where strict parametric assumptions may be fragile.
