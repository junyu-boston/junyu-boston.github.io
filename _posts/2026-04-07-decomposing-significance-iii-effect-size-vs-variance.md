---
title: "Decomposing Significance III: Effect Size vs Variance — What Actually Drove Your Results?"
date: 2026-04-07
categories: [Statistics]
tags: [effect-size, variance, diagnostics, count-data, signal-to-noise]
math: true
---

*This is Part III of the **Decomposing Significance** series: [Part I](/posts/decomposing-significance-i-anatomy-of-a-test-statistic/) | [Part II](/posts/decomposing-significance-ii-borrowing-strength/) | Part III (this article) | [Part IV](/posts/decomposing-significance-iv-permutation-and-the-honesty-panel/)*

---

## The decomposition

Parts I and II established the machinery. Every differential test is a ratio of estimated effect to estimated noise. Empirical Bayes shrinkage stabilizes the noise estimates but can also compress them systematically. The result: the number of significant features is a function of both the effects (numerator) and the variance estimates (denominator), and a pipeline that reports "2,000 significant features" tells you nothing about which component produced that number.

This part develops the diagnostic tools to separate the two.

The goal is concrete: after running these diagnostics, you should be able to say one of three things:

1. "Significance is driven by effect sizes — the effects are large and the variance structure is unremarkable."
2. "Significance is driven by variance compression — the effects are small and the variance estimates are unusually tight."
3. "Both components contribute — the effects are moderate and the variance estimates are moderately compressed."

Each of these leads to a different interpretation. Only (1) supports strong claims about the underlying process. (2) means the significant features are likely artifacts of the estimation procedure. (3) requires judgment about which component dominates.

## Diagnostic 1: The effect size distribution

The simplest and most powerful diagnostic is the distribution of absolute effect sizes.

For every feature $$j$$, you have an estimated log fold change $$\hat{\beta}_j$$. Plot the distribution of $$|\hat{\beta}_j|$$ for:

- All features (the baseline)
- Significant features only (the subset selected by FDR)

### What effect-driven significance looks like

If significance is driven by large effects, the distribution of $$|\hat{\beta}_j|$$ for significant features should be shifted right relative to the full distribution. The significant features are the ones with the largest effects. The median $$|\hat{\beta}_j|$$ for the significant set should be substantially larger than the median for the full set.

```text
Effect-driven:

All features:    |▓▓▓▓▓▒▒░░ · · ·|
Significant:     |  · ·▒▒▓▓▓▓▓▓▒▒|
                 0              |β|
```

### What variance-driven significance looks like

If significance is driven by variance compression, the distribution of $$|\hat{\beta}_j|$$ for significant features may be barely shifted — or not shifted at all. The significant features are not the ones with the largest effects. They are the ones with the smallest variance estimates. The median $$|\hat{\beta}_j|$$ for the significant set may be close to the overall median.

```text
Variance-driven:

All features:    |▓▓▓▓▓▒▒░░ · · ·|
Significant:     |▓▓▓▓▒▒░ · · · · |
                 0              |β|
```

The red flag: if the majority of significant features have $$|\hat{\beta}_j| < 0.5$$ (on a log₂ scale), you are promoting tiny effects to statistical significance. This is not inherently wrong — tiny effects can be real — but it should be a deliberate choice, not an accident of the variance estimation procedure.

### Comparing across experiments

This diagnostic becomes especially powerful when comparing two experimental conditions. If Condition A yields 2,000 significant features and Condition B yields 500, plot the $$|\hat{\beta}_j|$$ distributions side by side.

If the distributions are essentially identical — same median, same shape, same tails — but the significance counts differ dramatically, the difference is not in the effects. It is in the variance estimates.

> "The experiment increases statistical sensitivity without increasing effect size."

That sentence is a precise diagnosis. It does not claim the results are wrong. It says the results are driven by the denominator.

## Diagnostic 2: The mean-variance comparison

Part I introduced the mean-variance plot. Here we use it as a comparative tool.

Plot the mean-variance relationship for each condition separately:

- x-axis: $$\log_{10}(\bar{\mu}_j)$$
- y-axis: $$\log_{10}(\hat{\sigma}^2_j)$$

Overlay the fitted trends for each condition.

### What to look for

**Global downward shift.** If one condition's variance curve sits uniformly below the other's, that condition has tighter noise across all features. Any differential test between these conditions and a shared control will produce more significant features for the low-variance condition — not because the effects are larger, but because the noise floor is lower.

**Dispersion compression.** If one condition's fitted dispersion trend is flatter or narrower than the other's, shrinkage is pulling dispersions into a tighter range. This reduces the SE for most features and inflates test statistics.

**Trend shape change.** If the shape of the mean-variance relationship differs between conditions (e.g., one is linear on the log-log scale, the other is curved), pooled-trend shrinkage is fitting a compromise that describes neither condition accurately.

Each of these patterns indicates that the variance structure — not the effect structure — differs between conditions. If you observe any of them and also see different significance counts, the variance mechanism is a primary driver.

## Diagnostic 3: p-value vs effect size coupling

This is the most intuitive diagnostic for non-statisticians.

Plot:

- x-axis: $$|\hat{\beta}_j|$$ (absolute effect size)
- y-axis: $$-\log_{10}(p_j)$$ (significance)

This is the familiar volcano plot, but read diagnostically rather than as a feature selection tool.

### Effect-driven significance: diagonal structure

When significance is driven by effect sizes, large effects produce small p-values. The plot shows a clear diagonal trend: as $$|\hat{\beta}_j|$$ increases, $$-\log_{10}(p_j)$$ increases monotonically.

```text
Effect-driven:

-log₁₀(p)  │         ·  ·
            │       · ·
            │     ··
            │   ··
            │ ··
            │··
            └──────────────── |β|
```

### Variance-driven significance: vertical banding

When significance is driven by variance compression, you see a strong vertical band: many features with tiny $$|\hat{\beta}_j|$$ but extreme $$-\log_{10}(p_j)$$ values. The coupling between effect size and significance is broken.

```text
Variance-driven:

-log₁₀(p)  │·
            │·
            │·    ·
            │··  ·
            │··· ·
            │····· ·
            └──────────────── |β|
```

The vertical band means: "These features are highly significant not because their effects are large, but because their variance estimates are tiny." The test statistic is inflated by the denominator, not the numerator.

If you see vertical banding in a volcano plot, that is the single strongest visual evidence that variance is driving your results.

## Diagnostic 4: Signal-to-noise ratio decomposition

This diagnostic makes the decomposition quantitative.

For each feature, compute:

$$
\text{SNR}_j = \frac{|\hat{\beta}_j|}{\hat{\sigma}_j}
$$

where $$\hat{\sigma}_j$$ is the estimated standard deviation (square root of the variance estimate used in the test).

Now decompose the distribution of SNR into its two components:

1. **Distribution of $$|\hat{\beta}_j|$$** (effect sizes)
2. **Distribution of $$\hat{\sigma}_j$$** (noise estimates)

If you are comparing two conditions and one produces more significant features, ask:

- Did the median $$|\hat{\beta}_j|$$ increase? (Effects got larger → effect-driven)
- Did the median $$\hat{\sigma}_j$$ decrease? (Noise got smaller → variance-driven)
- Which component shifted more?

An even simpler version: compute the **median within-condition variance** across all features for each condition. If Condition A's median variance is 40% lower than Condition B's, you have a 40% compression of the denominator. That alone can explain a dramatic increase in significance counts, with no change in effects.

$$
\text{median}(\hat{\sigma}^2_A) \ll \text{median}(\hat{\sigma}^2_B) \implies \text{DE}_A \gg \text{DE}_B
$$

regardless of effect sizes.

## Diagnostic 5: Significance count vs within-group variance

This is the operational proof.

Across multiple experiments (or across subsets of features within a single experiment), plot:

- x-axis: mean within-group variance (averaged over features)
- y-axis: number of significant features

If significance is variance-driven, you will observe:

$$
N_{\text{sig}} \propto \frac{1}{\bar{\sigma}^2}
$$

This is a direct consequence of the test statistic structure. When the denominator shrinks uniformly, the number of features whose test statistic exceeds the threshold increases — and the relationship is approximately inversely proportional.

Plot it. If the data falls on a hyperbola, your significance counts are controlled by the noise floor, not by the effects.

## Diagnostic 6: Threshold sensitivity

Change the significance threshold and observe what happens.

- FDR 5% → FDR 10%
- Add a minimum $$|\hat{\beta}_j|$$ cutoff of 0.5 or 1.0
- Remove the $$|\hat{\beta}_j|$$ cutoff if one was present

### Variance-driven significance is threshold-sensitive

If the majority of significant features have small effects, relaxing the FDR threshold adds a flood of additional features — all with similarly small effects. The significance count is highly sensitive to threshold choice because the features are clustered near the decision boundary.

If you also add a modest effect size floor (e.g., $$|\hat{\beta}_j| > 1$$), the number of significant features may collapse. A result that goes from 2,000 features at FDR 5% to 150 features at FDR 5% with $$|\hat{\beta}_j| > 1$$ is telling you: most of the 2,000 had negligible effect sizes.

### Effect-driven significance is threshold-robust

If significance is driven by large effects, moderate changes to FDR or effect size thresholds have modest impact. The significant features are well-separated from the decision boundary, and the counts are stable under perturbation.

## The summary table

Putting it all together:

| Diagnostic | Effect-driven | Variance-driven |
|:-----------|:-------------|:----------------|
| $$\|\hat{\beta}\|$$ distribution | Shifts right for significant features | Static or barely shifts |
| Mean-variance trend | Stable across conditions | Condition-specific shifts |
| Volcano plot | Diagonal (coupled) | Vertical banding (decoupled) |
| SNR decomposition | $$\|\hat{\beta}\|$$ increases | $$\hat{\sigma}$$ decreases |
| $$N_{\text{sig}}$$ vs variance | Flat / uncorrelated | Inversely proportional |
| Threshold sensitivity | Low (robust) | High (fragile) |

If three or more diagnostics point to variance-driven significance, the inference is clear: the significant feature count reflects the noise estimation procedure, not the strength of the underlying effects.

This does not mean the results are wrong. It means they need to be interpreted as "we detected small effects because our noise estimates were tight" rather than "we detected strong effects." These are fundamentally different claims with different downstream implications.

## A worked example (synthetic data)

To make this concrete, consider a simulated scenario.

Generate $$p = 10{,}000$$ features under two conditions (this could be word counts across documents, species abundances across sites, defect categories across production runs — the math is identical):

- **Condition A**: 5 replicates, within-group standard deviation = 0.8
- **Condition B**: 5 replicates, within-group standard deviation = 0.4

Both conditions share the **same true effect sizes**: 95% of features have $$\beta_j = 0$$ (null), 5% have $$\beta_j \sim N(0, 0.3^2)$$ (small true effects).

Run the same differential testing pipeline on both. Expected results:

- Condition B produces roughly 2-4x more significant features than Condition A
- The effect size distributions are identical (by construction)
- The mean-variance plot for B sits below A
- The volcano plot for B shows vertical banding; A shows a more diagonal structure
- $$N_{\text{sig}}$$ vs $$\bar{\sigma}^2$$ falls on a hyperbola

Every diagnostic confirms: the significance difference is entirely variance-driven. No real difference in effects. Just noise.

This is not a pathological scenario. It is the default behavior of these methods when within-group variance differs across conditions.

## The uncomfortable implication

When someone reports "Condition A produced 2,000 significant features while Condition B produced 500," the instinctive interpretation is "Condition A has stronger or more widespread effects."

These diagnostics may reveal the opposite: Condition A simply had tighter noise, and the effect structure is indistinguishable between conditions. The significance count is measuring the precision of the experiment, not the magnitude of the effects.

This matters because downstream decisions — which condition to pursue, which results to follow up on, how to rank experimental or design variants — are often based on significance counts. If those counts are variance-driven, the ranking is an artifact. The "better" condition is just the quieter one.

> "The number of significant features is a measure of statistical sensitivity, not of effect magnitude."

This is not a philosophical objection. It is a quantitative claim that can be tested with the diagnostics above.

---

*Previous: [Part II — Borrowing Strength: Empirical Bayes Shrinkage for Variance](/posts/decomposing-significance-ii-borrowing-strength/)*

*Next: [Part IV — Permutation Testing and the Honesty Panel](/posts/decomposing-significance-iv-permutation-and-the-honesty-panel/)*
