---
title: "Imbalanced Regression: Why QSAR Models Miss the Rare Active Compounds"
date: 2026-04-18
categories: [Machine Learning]
tags: [imbalanced-regression, qsar, drug-discovery, model-evaluation, rare-events]
math: true
---

## The model can look accurate while missing the molecules you actually care about

In classification, imbalance is easy to spot. You have very few positives, many negatives, and standard metrics can lie to you.

In regression, the same problem is easier to miss.

Suppose you are building a QSAR model for activity prediction. Most compounds in the dataset are mediocre or inactive. A small fraction are genuinely strong actives. The target is continuous, so nothing in the setup forces you to notice that the distribution is heavily skewed toward the boring middle.

Then the model trains, the RMSE looks decent, the $R^2$ is acceptable, and the scatter plot seems respectable. But when you use the model to prioritize compounds, the top of the ranked list is disappointing. The rare high-activity molecules were exactly the part of the distribution the model learned least well.

That is the core problem of **imbalanced regression**: the response variable is continuous, but the region you care about occupies very little probability mass.

## What imbalance means in regression

In classification, imbalance means one class is rare.

In regression, imbalance means some target ranges are rare and others are common **and** the rare region carries disproportionate practical value. Formally, the marginal distribution of the response, $p(y)$, is highly uneven, while the utility of prediction is asymmetric across $y$. In QSAR, the high-activity tail often contains only a small fraction of compounds, while the low-to-moderate activity range dominates the sample.

If the practical objective is hit finding, lead prioritization, or identifying the top-scoring candidates for follow-up, that tail is not just another part of the distribution. It is the operational target.

This is why standard regression training can go wrong even when it is mathematically behaving exactly as designed.

## Why ordinary regression losses struggle

Most standard regression objectives are average-error objectives. Mean squared error minimizes:

$$
\mathcal{L}_{\text{MSE}} = \frac{1}{n} \sum_{i=1}^{n} (y_i - \hat{y}_i)^2
$$

Every observation contributes to the average. If 90% of the dataset lives in the mediocre range, then most gradient updates are driven by that region. Rare high-activity compounds can still produce large residuals under squared loss, but they are often too sparsely supported, too noisy, and too few in number to define the fit on their own.

The model is therefore rewarded for being broadly right where the data are dense, even if it is systematically wrong where the scientific value lives.

That produces several practical failure modes.

## Failure mode 1: regression to the middle

The most common failure is simple shrinkage toward the dense region of the target distribution.

If the model sees mostly middling activities, it learns that predicting something near the center is usually safe. Extreme values are expensive to predict incorrectly, and there are too few of them to dominate the objective. The result is underprediction of rare strong actives and overprediction of rare strong inactives.

In QSAR language: the model learns the chemistry of the average compound better than the chemistry of the exceptional compound.

This is not just a calibration issue. It changes ranking. A truly strong active may be pushed down into the crowded middle, where it becomes difficult to distinguish from hundreds of mediocre molecules.

## Failure mode 2: good RMSE, bad prioritization

A regression model can have perfectly decent global metrics and still be operationally weak.

Why? Because RMSE and $R^2$ average over the full target distribution. They answer questions like:

- how close are predictions on average?
- how much variance is explained globally?

Those are not the same as:

- are the strongest actives ranked near the top?
- does the top 1% of predictions contain most of the useful molecules?
- when the model predicts a very high value, is that signal trustworthy?

If your workflow reviews only the top 50 or top 500 compounds, then the quality of the middle of the distribution is much less important than the quality of the right tail.

This is the regression version of the ROC-versus-PR problem in classification: a global metric can look healthy while the top-of-list behavior is exactly what is broken.

## Failure mode 3: the tail is noisy, so the model avoids it

Rare high-response regions are often noisier than the center of the distribution.

In drug discovery this is common. Potent compounds may come from narrower chemistry, assay variance can be heteroscedastic, and the high-activity tail may contain more measurement uncertainty, fewer analogues, and less local support in feature space. So the region you care about is not only small — it is also statistically harder.

That makes the optimization problem even more biased toward the middle. The dense region is both more common and easier to fit.

## Failure mode 4: random train/test splits can hide the problem

If the target tail is rare, a random split may place very few strong actives into validation or test. Then the evaluation itself becomes unstable.

You can get lucky and report a flattering number because the hardest tail cases were barely represented. Or you can get unlucky and draw conclusions from a handful of compounds. Either way, the estimate of model quality in the region that matters is noisy.

In QSAR, this gets worse if analogues of the same chemical series are split across train and test. Then the model may appear to handle the tail well because it saw close neighbors of those actives during training.

## Failure mode 5: acquisition bias compounds the target imbalance

There is often a second imbalance layered on top of the first.

Medicinal chemistry programs do not sample chemistry uniformly. They enrich around promising series, discard unpromising directions, and often assay compounds only after earlier filtering. So the training set is not merely a skewed draw from activity values; it is also a biased draw from chemical space.

Now the model faces two distortions at once:

- the activity distribution is tail-sparse
- the chemical support for that tail is uneven and often clustered

That combination can make the model look better within familiar series and worse on the genuinely novel actives you hope it will discover.

## What to do instead

There is no single fix. Imbalanced regression is not one bug. It is a mismatch between the learning objective and the scientific objective.

The main rule is straightforward: **make the model pay more attention to the part of the response distribution you actually care about.**

### 1. Reweight the loss by target relevance

The simplest idea is to assign larger weights to rare or important target ranges:

$$
\mathcal{L}_{\text{weighted}} = \frac{1}{n} \sum_{i=1}^{n} w(y_i) (y_i - \hat{y}_i)^2
$$

where $w(y)$ increases in the high-activity tail.

This makes the rare actives matter more during optimization. Instead of treating every residual equally, you encode the fact that missing a potent compound is worse than being a bit off on a mediocre one.

The key difficulty is choosing $w(y)$ sensibly. If you weight the tail too aggressively, the model can overfit a tiny number of noisy compounds. If you weight it too weakly, nothing changes.

### 2. Use relevance functions rather than raw frequency alone

In imbalanced regression, the important region is not always the rarest region. It is the region with the highest practical value.

That is why many imbalanced-regression methods define a **relevance function** $\phi(y) \in [0,1]$ rather than just inverse-frequency weights. In QSAR, relevance usually increases with activity because the upper tail is what drives follow-up experiments. But the same framework works whenever domain value is asymmetric.

This is often a better mental model than "rebalance the histogram." You are not fixing the target distribution for fairness. You are telling the model which errors are scientifically expensive.

### 3. Resample the rare-response region carefully

Another family of methods changes the training distribution itself.

- oversample rare high-response observations
- undersample the dominant middle region
- synthesize tail observations with methods such as SMOTER or SMOGN

These methods can help, especially with tree ensembles or local learners, but they need care. Synthetic tail samples are only useful if interpolation in feature space remains chemically meaningful. In QSAR, naive interpolation can create descriptor-space samples that do not correspond to valid chemistry or that distort the local structure-activity relationship.

So resampling is not free signal. It is a way of changing optimization pressure, not a way of inventing truth.

### 4. Optimize ranking, not only numeric accuracy

If the workflow is ultimately about prioritization, then pure point prediction is not the whole problem.

You may care more about whether the model places potent compounds above weak ones than whether every predicted pIC50 is numerically precise. In that case, pairwise or listwise ranking objectives can be more aligned with the decision process than plain MSE.

This is especially true when the downstream question is: which compounds do we synthesize next?

A model that gets the ordering right in the upper tail may be more useful than a model with slightly lower RMSE but weaker prioritization.

### 5. Use a two-stage model when the tail is the real target

Sometimes the cleanest solution is not one regression model.

Instead, separate the problem into stages:

1. classify whether a compound exceeds a scientifically meaningful threshold
2. regress within the promising region or re-rank the shortlisted compounds

This mirrors a common discovery workflow. First ask: is this likely to be interesting at all? Then ask: among the interesting ones, which are strongest?

That decomposition often matches the data geometry better than forcing one global regressor to solve everything at once.

### 6. Evaluate the tail explicitly

If you only report RMSE and $R^2$, you are asking to miss the problem.

Add metrics that expose right-tail behavior. Depending on the workflow, that can include:

- RMSE restricted to the high-response region
- weighted RMSE using the same relevance function used in training
- Spearman correlation within the top quantile
- top-$K$ mean observed activity
- fraction of actives recovered in the top $K$ after thresholding the response
- enrichment or lift relative to random selection

This is the regression analogue of switching from ROC alone to PR-style metrics when positives are rare.

### 7. Split data in a way that tests the real problem

For QSAR, evaluation should usually respect chemistry as well as target balance.

That often means scaffold-aware or series-aware splitting, not just random splitting. And if the high-activity tail is sparse, you should inspect how many tail cases appear in each fold rather than assuming cross-validation will average the problem away.

The goal is not to guarantee perfect balance. The goal is to make sure the test set actually interrogates the rare region you claim to care about.

### 8. Model uncertainty matters more in the tail

When support is sparse, point predictions are less trustworthy. That is exactly where uncertainty estimates become valuable.

If two compounds have similar predicted activity but one sits in a dense familiar region and the other sits in a sparse extrapolative region, those predictions should not be treated equally. Ensembles, conformal prediction, Bayesian approximations, or even simple variance estimates across models can help flag when a high predicted value is mostly optimism.

In active learning settings, this can be especially powerful: one strategy is to query compounds that are predicted high and uncertain, not just high.

## The honest way to think about it

Imbalanced regression is not just "class imbalance for continuous labels." The geometry is more subtle.

The failure is not that the model ignores a minority class. The failure is that average-error training and average-error evaluation are often misaligned with a tail-focused scientific objective.

In QSAR, that objective is usually not to predict every mediocre compound as accurately as possible. It is to find the rare compounds worth caring about.

That means the right questions are:

- where is the scientifically valuable part of the response distribution?
- how much training signal reaches that region?
- does the objective emphasize it?
- does the evaluation reveal whether it is being modeled well?

If the answer to those questions is "not really," then a decent RMSE is not reassurance. It is camouflage.

## The practical recipe

If I were building a QSAR model on a heavily skewed activity distribution, I would default to this workflow:

1. Inspect the target distribution first, especially the upper tail and any assay censoring.
2. Define the scientifically meaningful high-response region explicitly.
3. Use scaffold-aware evaluation and make sure tail cases are represented in validation.
4. Report at least one tail-focused metric in addition to RMSE or $R^2$.
5. Try relevance-weighted losses or a ranking-aware objective.
6. Consider a two-stage model if the top tail is the real decision target.
7. Check uncertainty before trusting very high predictions in sparse regions.

That workflow is not mathematically fancy. It is just honest about what the model is being asked to do.

## The bottom line

When the response distribution is imbalanced, ordinary regression tends to spend most of its statistical budget where the data are dense, not where the value is high.

For QSAR and related drug-discovery problems, that is exactly backwards. The mediocre compounds dominate the dataset. The rare active compounds dominate the decision.

If you train and evaluate as though those two facts are the same thing, the model will often look better than it actually is.

## References

- [Branco, Torgo, and Ribeiro, 2017](https://doi.org/10.1145/3122009) — survey of predictive modeling under imbalanced distributions
- [Torgo et al., 2013](https://doi.org/10.1007/978-3-642-40669-0_5) — SMOTE for regression
- [Ribeiro, Moniz, and Torgo, 2020](https://doi.org/10.1007/978-3-030-45716-9_9) — imbalanced regression and utility-based learning
