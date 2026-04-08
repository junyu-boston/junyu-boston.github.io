---
title: "Decomposing Significance I: The Anatomy of a Test Statistic in Count Data"
date: 2026-04-06
categories: [Statistics]
tags: [negative-binomial, glm, count-data, hypothesis-testing, mean-variance]
math: true
---

*This is Part I of the **Decomposing Significance** series: Part I (this article) | [Part II](/posts/decomposing-significance-ii-borrowing-strength/) | [Part III](/posts/decomposing-significance-iii-effect-size-vs-variance/) | [Part IV](/posts/decomposing-significance-iv-permutation-and-the-honesty-panel/)*

---

## The question nobody asks

When a statistical pipeline reports 2,000 significant features out of tens of thousands, most practitioners accept the number at face value. The FDR is controlled at 5%. The method is peer-reviewed. The software is mature. So 2,000 it is.

But there is a question that almost never gets asked:

> Why are those 2,000 features significant — because the effects are large, or because the variance estimates are small?

This distinction matters enormously. A feature can cross a significance threshold because something real happened to it, or because the noise estimate shrank enough to make an ordinary fluctuation look extreme. In small-sample, high-dimensional settings, the second mechanism dominates far more often than most people realize.

This series is about learning to tell the difference.

## Everything is a ratio

Strip away the software, the model-specific notation, the pipeline complexity. At the core of every differential test on count data, you will find a structure that looks like this:

$$
\text{test statistic} \approx \frac{\hat{\beta}}{\widehat{\text{SE}}(\hat{\beta})}
$$

where $$\hat{\beta}$$ is the estimated effect size (typically a log fold change) and $$\widehat{\text{SE}}$$ is its estimated standard error.

This is a Wald-type statistic. Likelihood ratio tests and score tests have different algebra, but the same fundamental sensitivity: significance depends on the ratio of signal to estimated noise, not on signal alone.

That ratio is the key to everything that follows.

If the numerator grows — larger effects, real shifts — the statistic gets larger and significance increases. This is what we want. But the statistic also grows when the denominator shrinks — when variance is underestimated, or when shrinkage compresses dispersion estimates toward a too-narrow global trend. This produces significant features without meaningful effects.

Both mechanisms produce the same output: a list of significant features with controlled FDR. The pipeline cannot tell you which mechanism produced its results. You have to figure that out yourself.

## Why count data makes this worse

Continuous measurements (heights, voltages, temperatures) have a useful property: the variance and the mean are, to a first approximation, independent. You can estimate each one separately and the errors do not compound in systematic ways.

Count data breaks this. In count data, variance is a function of the mean. Always.

The simplest count model — the Poisson — enforces this rigidly:

$$
Y \sim \text{Poisson}(\mu) \implies \text{Var}(Y) = \mu
$$

The variance equals the mean. No free parameter, no flexibility. If the mean is 100, the variance is 100. If the mean is 10, the variance is 10.

This is almost never realistic for high-dimensional count data in practice — whether the counts represent page views per URL, species abundances per site, word frequencies per document, or defect counts per product line. Real counts are **overdispersed**: the observed variance exceeds the Poisson prediction, often by a large factor. Counts of 100 might have a variance of 500 or 5,000 rather than 100. The Poisson model has no way to accommodate this.

## The negative binomial fix

The negative binomial (NB) distribution adds a dispersion parameter $$\alpha$$ that decouples the variance from the rigid Poisson constraint:

$$
Y \sim \text{NB}(\mu, \alpha) \implies \text{Var}(Y) = \mu + \alpha \mu^2
$$

The variance now has two components:

- **$$\mu$$**: the Poisson contribution (shot noise, irreducible)
- **$$\alpha \mu^2$$**: the overdispersion term, scaled by the dispersion parameter $$\alpha$$

When $$\alpha = 0$$, this collapses to the Poisson. As $$\alpha$$ grows, variance grows quadratically with the mean rather than linearly. This quadratic mean-variance relationship is the signature of negative binomial count data.

Here is why this matters for differential testing. In a generalized linear model (GLM) for NB counts, the log-link regression is:

$$
\log \mu_{ij} = \mathbf{x}_i^{\top} \boldsymbol{\beta}_j
$$

where $$i$$ indexes samples and $$j$$ indexes features. The coefficient $$\beta_j$$ for a group indicator is the log fold change for feature $$j$$. The significance test for $$\beta_j$$ depends on the estimated dispersion $$\hat{\alpha}_j$$.

The standard error of $$\hat{\beta}_j$$ is approximately:

$$
\widehat{\text{SE}}(\hat{\beta}_j) \propto \sqrt{\frac{1 + \hat{\alpha}_j \bar{\mu}_j}{n \bar{\mu}_j}}
$$

(The exact form depends on the design matrix and estimation method, but the dependence on $$\hat{\alpha}_j$$ is the critical piece.)

This means:

- **Overestimate $$\alpha_j$$** → SE is inflated → test statistic shrinks → feature is not significant (conservative)
- **Underestimate $$\alpha_j$$** → SE is deflated → test statistic inflates → feature is significant (anti-conservative)
- **Shrink all $$\alpha_j$$ toward a global trend** → every feature's SE is pulled toward the trend → significance is driven by the trend, not by individual feature behavior

That third case is exactly what empirical Bayes shrinkage does. And it is the default behavior of every major pipeline for this kind of analysis.

## The mean-variance plot: your first diagnostic

Before running any differential test, plot the mean-variance relationship. For every feature:

- x-axis: $$\log_{10}(\bar{\mu}_j)$$ (mean count across samples)
- y-axis: $$\log_{10}(\hat{\sigma}^2_j)$$ (observed variance)

For Poisson data, every point lies on the diagonal $$\sigma^2 = \mu$$. For NB data, points lie above the diagonal, and the vertical distance from the Poisson line is a visual measure of overdispersion.

Now plot the **fitted trend** — the curve that the model uses to estimate dispersion. In most pipelines, this is a smooth function of the mean. The gap between the observed scatter and the fitted trend tells you how much information each feature's variance estimate is borrowing from the global trend versus retaining from its own data.

If the fitted trend is tight and the scatter is wide, the model is imposing a lot of shrinkage. Features with unusually high or low variance are being pulled hard toward the trend. This can be appropriate — or it can be masking real heterogeneity in variance structure. The plot will not resolve that ambiguity on its own, but it will show you how much trust the model places in global versus local information.

This is the single most important diagnostic plot in count-based differential testing. Most practitioners generate it, glance at it, and move on. We will come back to it in Part III with sharper tools.

## What the GLM actually estimates

It is worth being precise about what a negative binomial GLM is doing, because the precision reveals the vulnerability.

You have $$n$$ samples (say, 6-12) and $$p$$ features (often thousands to hundreds of thousands). For feature $$j$$:

1. **Model**: $$Y_{ij} \sim \text{NB}(\mu_{ij}, \alpha_j)$$ with $$\log \mu_{ij} = \mathbf{x}_i^{\top} \boldsymbol{\beta}_j$$
2. **Estimate $$\boldsymbol{\beta}_j$$**: via maximum likelihood or iteratively reweighted least squares
3. **Estimate $$\alpha_j$$**: separately, because it does not enter the mean model — it enters the variance model
4. **Test**: construct a test statistic for $$\beta_{j,\text{treatment}} = 0$$ using $$\hat{\beta}_j$$ and $$\widehat{\text{SE}}(\hat{\beta}_j)$$

Steps 1, 2, and 4 are standard GLM machinery. Step 3 is where the difficulty lives.

With 3-5 replicates per condition, the per-feature estimate of $$\alpha_j$$ has essentially no precision. You are estimating a variance parameter from a handful of observations. The sampling distribution of $$\hat{\alpha}_j$$ is extremely wide. Some features will have their variance dramatically overestimated, others dramatically underestimated, purely by chance.

This is the small-n crisis that motivates everything in Part II: empirical Bayes shrinkage.

## The two ways significance inflates

We can now state the problem precisely.

**Mechanism 1: Real effects.** The condition genuinely shifts $$\mu$$ for many features. $$\hat{\beta}_j$$ is large. The test statistic is large because the numerator is large. Significance is driven by whatever real-world process generated the data.

**Mechanism 2: Variance compression.** The condition does not shift $$\mu$$ meaningfully, but something about the experimental setup reduces the observed variance. Maybe one condition has less technical noise. Maybe the samples are more homogeneous. Maybe the variance estimator is shrinking dispersions too aggressively. Whatever the cause, $$\widehat{\text{SE}}(\hat{\beta}_j)$$ shrinks, the test statistic inflates, and features are called significant even though their effects are negligible.

The pipeline produces a list of significant features in both cases. The FDR correction controls the proportion of false positives under the model's assumptions, but it does not distinguish between these two mechanisms. If the model's variance estimates are systematically too low, FDR correction faithfully reports 5% false discoveries *relative to the deflated variance model* — which may have nothing to do with the actual false discovery rate.

## Why this matters in practice

This is not a theoretical concern. It is an operational one.

If you are comparing two experimental conditions and reporting "Condition A produces 2,000 significant features while Condition B produces 500," the natural interpretation is that Condition A has stronger or more widespread effects.

But if Condition A happens to have tighter within-group variance — less technical noise, more homogeneous samples, or simply different variance structure — the count difference could be entirely variance-driven. The effect size distributions might be identical between conditions. The significance counts diverge because the denominator changed, not the numerator.

In small-sample experiments — common in ecology, clinical studies, A/B tests with limited cohorts, and many scientific fields — this is not a rare edge case. It is the default failure mode.

## Where this series goes

The rest of this series builds the tools to decompose significance into its components:

- **Part II** covers empirical Bayes shrinkage — how borrowing variance information across features stabilizes inference, and the failure modes it introduces
- **Part III** develops the diagnostic framework: effect size distributions, p-value vs effect size coupling, SNR decomposition, and the summary table that separates effect-driven from variance-driven significance
- **Part IV** introduces permutation testing as ground truth, rank stability analysis, and a one-page diagnostic panel that answers "should I trust these results?"

The goal is not to argue that differential testing is broken. It is not. The goal is to give practitioners the diagnostic tools to know *why* their features are significant — and to recognize when significance is an artifact of variance estimation rather than a reflection of real effects.

---

*Next: [Part II — Borrowing Strength: Empirical Bayes Shrinkage for Variance](/posts/decomposing-significance-ii-borrowing-strength/)*
