---
title: "Differential Expression as a Likelihood-Ratio Test: The Gaussian View"
date: 2026-04-25
categories: [Computational Biology, Statistics]
tags: [differential-expression, likelihood-ratio-test, gaussian-models, t-statistic, rna-seq, mahalanobis-distance]
description: "A theoretical explanation of differential expression as a likelihood-ratio test, why Gaussian assumptions recover the familiar t-statistic, and why low-variance genes are easier to call significant."
excerpt: "Differential expression is not mainly about raw change. It asks whether a model with different means explains the data enough better than a constant-mean model, once variance is taken seriously."
math: true
---

*Differential expression is not mainly a contest of raw fold changes. At its core, it is a model comparison: does the data look materially more likely if I let the mean move between conditions?*

This connects several ideas that are often taught separately:

- differential expression
- likelihood ratios
- Gaussian models
- t-statistics
- Mahalanobis distance

They are all different views of the same statistical object. This explanation is intentionally theoretical. Real RNA-seq pipelines often use negative binomial models rather than Gaussian ones, but the Gaussian case makes the geometry easiest to see.

## One gene, two models

Take one gene $$g$$ measured across samples. After normalization and an appropriate transformation, suppose the observed values are reasonably modeled as

$$
x_{gi} \sim \mathcal{N}(\mu_g^{(k)}, \sigma_g^2)
$$

where $$k$$ indicates the condition, usually control or treatment.

The interpretation is simple:

- $$\mu_g^{(k)}$$ is the condition-specific mean
- $$\sigma_g^2$$ is the gene-specific variance

For differential expression, the question is whether the same mean can explain both groups.

### Null model

Under the null hypothesis,

$$
H_0: \mu_g^{(\mathrm{ctrl})} = \mu_g^{(\mathrm{treat})}
$$

there is one shared mean. The gene is not differentially expressed.

### Alternative model

Under the alternative,

$$
H_1: \mu_g^{(\mathrm{ctrl})} \neq \mu_g^{(\mathrm{treat})}
$$

the two groups get separate means. The gene is allowed to differ between conditions.

Differential expression is a comparison between a restricted model and a less restricted one.

## The core statistic is a likelihood ratio

Once the two models are written down, the natural question is:

> How much better does the data fit if I allow the mean to differ between conditions?

The likelihood ratio answers exactly that.

$$
\Lambda_g =
\frac{\max_{H_0} \mathcal{L}(x_g)}{\max_{H_1} \mathcal{L}(x_g)}
$$

or, more commonly,

$$
-2 \log \Lambda_g
$$

When this quantity is large, the unrestricted model explains the data much better than the restricted one. In plain language, the constant-mean model lost badly enough to a two-mean model that the data count as evidence for change.

## Why the Gaussian case turns into a t-statistic

Under Gaussian assumptions, the maximum-likelihood estimates are familiar:

- under $$H_0$$, the best mean is the pooled mean across both groups
- under $$H_1$$, the best means are the two group averages

When you take the log-likelihood under each model and subtract them, the algebra collapses to something proportional to

$$
\frac{(\bar{x}_{g,\mathrm{treat}} - \bar{x}_{g,\mathrm{ctrl}})^2}{\widehat{\sigma}_g^2}
$$

up to constants and degrees-of-freedom adjustments.

The likelihood-ratio statistic is just:

- squared mean shift in the numerator
- variance in the denominator

Take a square root, account for sample size and variance estimation, and you land in the familiar z-, t-, or F-test family.

$$
t_g \approx
\frac{\bar{x}_{g,\mathrm{treat}} - \bar{x}_{g,\mathrm{ctrl}}}{\widehat{\mathrm{SE}}(\Delta \mu_g)}
$$

So Gaussian-language differential expression is not a different idea. It is the same model comparison in a more recognizable form.

## This is why low-variance genes are easier to call significant

Once you see the likelihood-ratio form, one common pattern becomes unavoidable.

Suppose two genes have the same observed mean shift:

$$
\Delta \mu_g = \bar{x}_{g,\mathrm{treat}} - \bar{x}_{g,\mathrm{ctrl}}
$$

but one gene has much smaller variance.

Then, all else equal, the low-variance gene gets the larger statistic because

$$
\text{statistic} \propto \frac{(\Delta \mu_g)^2}{\sigma_g^2}
$$

That means:

- high-variance genes have broad likelihoods, so moderate shifts are not surprising
- low-variance genes have sharp likelihoods, so even modest shifts can be very surprising

This is sometimes described as a bias toward stable genes. In the Gaussian view, it is better understood as the model giving more weight to a mean shift when residual noise is smaller.

A small displacement inside a very tight distribution is more informative than the same displacement inside a noisy one. In count-based RNA-seq models, that intuition is still useful, but it is filtered through mean-variance coupling and dispersion estimation.

## Differential expression is a one-dimensional Mahalanobis distance

For a single gene, the basic differential-expression statistic looks like

$$
\frac{(\Delta \mu_g)^2}{\sigma_g^2}
$$

In multivariate statistics, the canonical variance-normalized distance is the Mahalanobis distance:

$$
(x - \mu)^\top \Sigma^{-1} (x - \mu)
$$

Gene-wise differential expression is the one-dimensional version.

That observation makes several things line up:

- PCA describes where variance lives
- Mahalanobis distance describes surprise relative to covariance structure
- differential expression asks whether the observed mean shift is surprising given the variance

In gene-wise DE, we usually ignore gene-gene covariance and work one feature at a time. But the conceptual bridge is real: a DE statistic is a variance-aware distance from a restricted model.

## Why shrinkage belongs in the story

In practice, the variance is not known. It has to be estimated.

That creates a problem, especially in small-sample settings. A noisy variance estimate can make the likelihood ratio unstable in either direction:

- underestimated variance makes significance look too easy
- overestimated variance makes real effects look smaller than they are

Empirical Bayes variance shrinkage pulls unstable gene-wise variance estimates toward a shared trend or prior distribution. In Gaussian workflows this shows up as moderated t-statistics. In count-based workflows the same logic appears through dispersion shrinkage or related regularization.

From the model-comparison point of view, shrinkage is not a side trick. It is part of keeping the denominator from becoming erratic enough to dominate the entire inference.

## The most common misunderstanding

The easiest mistake is to think differential expression answers this question:

> Which genes changed a lot?

That is incomplete.

The actual question is closer to this:

> Which genes show a change that is hard to explain under a constant-mean model, once uncertainty is taken into account?

Magnitude matters, but magnitude alone is not the test.

That is why a gene with a tiny fold change can still look convincingly different if its variance is extremely low. It is also why a dramatic-looking change in a noisy gene can fail to register as significant.

The test is measuring model-based surprise, not raw motion.

## The Gaussian model is only the transparent case

Real RNA-seq methods are usually built on count models, especially the negative binomial. That matters for calibration, mean-variance coupling, and the details of dispersion estimation.

But the conceptual backbone survives the change of distribution:

- one model says the mean is shared across conditions
- another model says the mean can differ
- inference is driven by how much better the less restricted model explains the data, relative to the noise structure

The Gaussian setting simply makes the reduction to a variance-normalized mean shift visible in one line.

## A clean way to remember it

If you want one compact mental model, use this:

> Differential expression is a likelihood-ratio test, and under Gaussian assumptions that likelihood ratio becomes a standardized mean shift.

Or even shorter:

> DE is not a fold-change filter. It is a surprise score under a probabilistic model.

Once that clicks, several ideas that are usually taught separately start to feel like one system.

## Takeaway

Differential expression, Gaussian test statistics, and Mahalanobis-style geometry are all versions of the same principle: signal only matters after it has been scaled by uncertainty.

That is why the t-statistic appears, why low-variance genes are easier to call, and why shrinkage matters. A DE result is best understood as evidence that the constant-mean model stopped being credible.

## Related posts

- [Decomposing Significance I: The Anatomy of a Test Statistic in Count Data](/posts/decomposing-significance-i-anatomy-of-a-test-statistic/)
- [Decomposing Significance III: Effect Size vs Variance](/posts/decomposing-significance-iii-effect-size-vs-variance/)
- [Beyond Correlation I: Mahalanobis PCA Geometry](/posts/beyond-correlation-i-mahalanobis-pca-geometry/)
