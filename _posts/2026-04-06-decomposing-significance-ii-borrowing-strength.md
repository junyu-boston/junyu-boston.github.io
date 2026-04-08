---
title: "Decomposing Significance II: Borrowing Strength — Empirical Bayes Shrinkage for Variance"
date: 2026-04-06
categories: [Statistics]
tags: [empirical-bayes, shrinkage, variance-estimation, count-data, negative-binomial]
math: true
---

*This is Part II of the **Decomposing Significance** series: [Part I](/posts/decomposing-significance-i-anatomy-of-a-test-statistic/) | Part II (this article) | [Part III](/posts/decomposing-significance-iii-effect-size-vs-variance/) | [Part IV](/posts/decomposing-significance-iv-permutation-and-the-honesty-panel/)*

---

## The small-n variance crisis

Part I established that every differential test in count data is a ratio: estimated effect divided by estimated standard error. The standard error depends on the dispersion parameter $$\alpha_j$$ for each feature $$j$$. And in a typical high-dimensional experiment — thousands to hundreds of thousands of features, a handful of replicates per condition — the per-feature estimate of $$\alpha_j$$ is almost worthless.

This is not a subtle point. With 3-5 observations per group, you are estimating a variance parameter from a handful of data points (or fewer, after accounting for the mean model). The sampling distribution of a variance estimate from 6 observations is enormously wide. Some features will have their dispersion overestimated by 5x. Others will have it underestimated by 5x. Purely by chance.

If you use these raw estimates directly in your test statistics, you get chaos: features with accidentally low variance estimates become significant regardless of effect size, and features with accidentally high variance estimates are missed regardless of effect size. The false discovery rate becomes uncontrollable — not because the FDR correction is wrong, but because the inputs to the correction are garbage.

This is the crisis that empirical Bayes shrinkage was designed to solve.

## The idea: borrow information across features

The key insight is that while any single feature's dispersion estimate is unreliable, you have thousands of them. Collectively, they contain a great deal of information about the overall variance structure.

Empirical Bayes shrinkage exploits this. The procedure, at a high level:

1. Estimate the dispersion $$\hat{\alpha}_j^{\text{raw}}$$ for each feature independently
2. Fit a global trend relating dispersion to the mean: $$\alpha(\mu)$$
3. Shrink each feature's estimate toward the trend:

$$
\hat{\alpha}_j^{\text{shrunk}} = w_j \cdot \hat{\alpha}_j^{\text{raw}} + (1 - w_j) \cdot \alpha(\bar{\mu}_j)
$$

where $$w_j \in [0, 1]$$ controls how much the feature retains its own estimate versus adopting the global trend.

Features with very few counts (and therefore very noisy dispersion estimates) are pulled strongly toward the trend ($$w_j$$ close to 0). Features with high counts and more reliable estimates retain more of their own information ($$w_j$$ closer to 1).

This is James-Stein shrinkage applied to variance estimation. The same principle that Stein demonstrated for means in 1956 — that shrinking toward a common value improves aggregate estimation error — applies to variances.

## What the shrinkage weight depends on

The weight $$w_j$$ is determined by two quantities:

- **Prior precision**: how tight the global distribution of dispersions is (tighter = more shrinkage)
- **Data precision**: how much information feature $$j$$'s own data provides about its dispersion (more replicates = less shrinkage)

Formally, in a parametric empirical Bayes framework:

$$
w_j = \frac{\text{prior precision}}{\text{prior precision} + \text{data precision}_j}
$$

With 3-5 replicates, the data precision is low for every feature, so shrinkage is heavy everywhere. With 20+ replicates, the data precision is high, and each feature retains most of its own estimate.

This is the right tradeoff when the prior is well-calibrated — when the global trend accurately describes how dispersion varies with the mean, and when individual features do not deviate from this trend in important, systematic ways.

When those conditions fail, shrinkage introduces its own problems. We will get to that.

## An alternative approach: precision-weighted linear models

There is a conceptually different strategy that avoids direct dispersion estimation for each feature. Instead of fitting a negative binomial GLM and then shrinking the dispersion, you can:

1. Transform the count data to approximate normality (using a mean-variance relationship to compute precision weights)
2. Fit an ordinary linear model to the transformed data, weighted by the precision
3. Use a moderated t-statistic that borrows information across features to stabilize the residual variances

The key step is the variance-stabilizing transformation. Given raw counts $$Y_{ij}$$, you compute:

$$
\tilde{Y}_{ij} = \log_2(Y_{ij} + 0.5)
$$

(or a more sophisticated transformation), then model the mean-variance trend of the log-transformed data and compute observation-level weights:

$$
w_{ij} = \frac{1}{\hat{f}(\bar{Y}_j)}
$$

where $$\hat{f}$$ is the fitted mean-variance function. Observations from high-variance features get downweighted; observations from low-variance features get upweighted.

The linear model is then:

$$
\tilde{Y}_{ij} = \mathbf{x}_i^{\top} \boldsymbol{\beta}_j + \epsilon_{ij}, \quad \epsilon_{ij} \sim N(0, \sigma_j^2 / w_{ij})
$$

The residual variance $$\sigma_j^2$$ is then moderated using an empirical Bayes procedure similar to the one described above — shrunk toward a global prior. The resulting moderated t-statistic has increased degrees of freedom, reflecting the information borrowed from other features.

This approach has a practical advantage: ordinary linear models are fast, well-understood, and numerically stable. The price is the approximation introduced by the transformation — you are fitting a linear model to data that is not truly linear, using weights that depend on estimated quantities.

Both approaches — direct NB GLM with dispersion shrinkage, and precision-weighted linear models with moderated statistics — use empirical Bayes to stabilize variance estimation. They differ in where the approximation lives: in the distributional model (NB vs normal) or in the transformation (raw counts vs log-counts with weights).

## What shrinkage does to the significance landscape

Here is the part that most practitioners do not think about carefully.

Before shrinkage, the per-feature dispersion estimates are scattered widely:

```text
Feature dispersion (raw):
  ○ ○     ○         ○   ○○     ○  ○    ○○  ○
  ─────────────────────────────────────────── → mean count
     ●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●  (fitted trend)
```

After shrinkage, every estimate is pulled toward the trend:

```text
Feature dispersion (shrunk):
  ●○●  ●○●●  ●○●●●○  ●●○●●●  ●●●○●●●  ●●●●
  ─────────────────────────────────────────── → mean count
     ●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●  (fitted trend)
```

The consequence for differential testing:

1. **Features that were spuriously significant** (because their raw dispersion was accidentally low) lose significance. Their shrunken dispersion is higher than the raw estimate, their SE increases, and their test statistic shrinks. This is the intended benefit.

2. **Features that were spuriously non-significant** (because their raw dispersion was accidentally high) gain significance. Their shrunken dispersion is lower than the raw estimate, their SE decreases, and their test statistic grows. This is also intended.

3. **Features with genuinely unusual dispersion** — those that truly differ from the global trend — are pulled toward a trend that does not describe them. If a feature has legitimately high variance (perhaps it is inherently noisy or responds heterogeneously across replicates), shrinkage will underestimate its variance and it may be called significant when it should not be.

Effect (3) is the failure mode. It is quiet, systematic, and almost never diagnosed.

## When shrinkage helps and when it hurts

### Shrinkage helps when

- **n is very small** (3-5 replicates): raw variance estimates are dominated by sampling noise
- **The global trend is a good description**: most features follow the same mean-variance relationship
- **Deviations from the trend are random**: features that deviate do so because of estimation noise, not because they have genuinely different variance structures

### Shrinkage hurts when

- **Subsets of features have systematically different variance**: for example, features with high variance in one condition and low variance in another. The global trend averages over this heterogeneity
- **Condition-specific variance exists**: the treatment changes the variance structure, not just the mean. Shrinkage toward a trend estimated from pooled data may not capture this
- **The trend is misspecified**: if the fitted mean-variance function has the wrong shape, shrinkage pulls everything toward the wrong target

The third case is particularly insidious because the mean-variance trend is itself estimated from the data. If the data contains a mixture of features with different variance behaviors, the trend will be a compromise — correct for no individual feature, but the average of several distinct regimes.

## The subtle problem: shrinkage and batch effects

Consider a practical scenario. You have two experimental conditions, A and B. Condition A was processed with one protocol and Condition B with another. The protocols produce different amounts of technical noise.

In Condition A, the within-group variance is high. In Condition B, the within-group variance is low. The mean-variance trend, fitted to pooled data, sits somewhere in the middle.

Now apply shrinkage:

- Features in Condition A are pulled *down* toward the trend (their raw variance was above the trend)
- Features in Condition B are pulled *up* toward the trend (their raw variance was below the trend)

The result: the shrunken dispersions for both conditions look more similar than the raw data warrants. The difference in variance structure — which is real and important — is compressed.

If you then test for differential effects, the test statistics are computed using variance estimates that partially erase the condition-specific noise difference. Depending on which direction the erasure goes, you may get inflated or deflated significance.

This is not a bug in the method. It is a fundamental limitation of borrowing information from a global trend when the data-generating process is heterogeneous across conditions.

## The moderated t-statistic and degrees of freedom

One concrete consequence of empirical Bayes shrinkage is the moderated t-statistic. Instead of the ordinary t-statistic:

$$
t_j = \frac{\hat{\beta}_j}{s_j \sqrt{v_j}}
$$

where $$s_j^2$$ is the residual variance for feature $$j$$ and $$v_j$$ is the unscaled standard error, the moderated version uses a posterior variance:

$$
\tilde{t}_j = \frac{\hat{\beta}_j}{\tilde{s}_j \sqrt{v_j}}
$$

where:

$$
\tilde{s}_j^2 = \frac{d_0 s_0^2 + d_j s_j^2}{d_0 + d_j}
$$

Here $$d_0$$ and $$s_0^2$$ are the prior degrees of freedom and prior variance (estimated from the global distribution), and $$d_j$$ and $$s_j^2$$ are the data degrees of freedom and observed variance for feature $$j$$.

The moderated statistic $$\tilde{t}_j$$ follows a t-distribution with $$d_0 + d_j$$ degrees of freedom — more than the ordinary t-statistic's $$d_j$$ degrees of freedom. The extra $$d_0$$ degrees of freedom come from the information borrowed from other features.

With small replicates per group (e.g., $$d_j = 4$$ for a two-group comparison with 3 per group), the ordinary t-distribution is extremely heavy-tailed: small denominator fluctuations produce enormous t-values. The moderated version might have $$d_0 + d_j = 20$$ or even $$d_0 + d_j = 50$$ effective degrees of freedom, producing a much tighter distribution. Extreme t-values are rarer, and the test is better calibrated.

But notice: the quality of this calibration depends entirely on the quality of the prior. If $$s_0^2$$ is a poor estimate of the typical variance, the extra degrees of freedom are borrowed from a model that does not match the data. The test becomes "well-calibrated under the wrong model" — which is a subtle and dangerous kind of miscalibration.

## What to look for in practice

Before trusting the output of any variance-shrinkage pipeline, check these diagnostics:

**1. Raw vs shrunken dispersion scatter plot.**
Plot $$\hat{\alpha}_j^{\text{raw}}$$ vs $$\hat{\alpha}_j^{\text{shrunk}}$$ for every feature. If the shrunken values are nearly constant (a flat line), shrinkage is extreme and individual feature behavior has been erased. If the scatter is preserved with moderate compression, shrinkage is doing its job.

**2. Mean-variance trend residuals.**
Plot the residuals of the mean-variance trend fit. If they show systematic structure — clusters of features consistently above or below the trend — the trend is misspecified and shrinkage is pulling features toward the wrong target.

**3. Condition-specific variance comparison.**
Split the data by condition and compute within-group variances separately. If the two conditions have visibly different variance structures, pooled-trend shrinkage may be inappropriate. Some pipelines allow condition-specific dispersion estimation; check whether yours does.

**4. Prior degrees of freedom.**
If the estimated $$d_0$$ is very large (say, > 30), the prior is dominating the posterior for most features. This means individual feature behavior matters very little. That is appropriate if the prior is correct and features genuinely share a common variance structure. It is dangerous if they do not.

## The connection to significance inflation

We can now connect the shrinkage machinery back to the question from Part I: why do 2,000 features come out significant?

If dispersion estimates are shrunk aggressively toward a low-variance trend, the standard errors of all effect estimates decrease. The test statistics inflate. Features with small but nonzero effects — effects that would be non-significant under honest per-feature variance estimation — cross the significance threshold.

The FDR correction does not save you here. FDR controls the *proportion* of false discoveries among the rejected features, given the model. If the model's variance estimates are systematically too low, the null distribution of the test statistic is too narrow, the p-values are too small, and FDR faithfully controls a proportion that is already miscalibrated.

The number of significant features goes up. It may go up a lot. And the entire increase can be driven by the shrinkage machinery rather than by the underlying effects.

Part III develops the diagnostic tools to detect exactly this situation.

---

*Previous: [Part I — The Anatomy of a Test Statistic in Count Data](/posts/decomposing-significance-i-anatomy-of-a-test-statistic/)*

*Next: [Part III — Effect Size vs Variance: What Actually Drove Your Results?](/posts/decomposing-significance-iii-effect-size-vs-variance/)*
