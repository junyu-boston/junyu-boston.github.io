---
title: "Classification Is Distribution Discrimination: ROC, MCC, and the Regression Bridge"
date: 2026-04-20
categories: [Machine Learning]
tags: [roc-auc, mcc, precision-recall, classification, regression, model-evaluation, statistical-testing]
math: true
---

*This article is a companion to [AUC-ROC vs AUC-PR](/posts/auc-roc-vs-auc-pr-what-changes-when-positives-are-rare/). That article covers when to use which metric. This one asks: what are all these metrics actually measuring, and why do they secretly connect to things that look nothing like classification?*

---

## The intuition that won't go away

If you have worked with ROC-AUC long enough, a feeling develops. AUC-ROC does not feel like a classification metric. It feels like you are measuring how well two distributions are separated.

That feeling is correct, and it is the key to understanding everything else in this article.

## ROC-AUC is a distribution discrimination statistic

Start from the probabilistic interpretation everyone learns: for continuous scores, AUC-ROC is the probability that a randomly drawn positive scores higher than a randomly drawn negative.

$$
\text{AUC} = P(S_+ > S_-)
$$

where $S_+$ is the score of a random positive and $S_-$ the score of a random negative.

If ties can occur, the exact form is:

$$
\mathrm{AUC} = P(S_+ > S_-) + \tfrac{1}{2} P(S_+ = S_-)
$$

Now notice what this actually says. You have two distributions — the score distribution of the positive class, $f_+(s)$, and the score distribution of the negative class, $f_-(s)$. AUC measures how separated they are.

A perfect classifier gives $P(S_+ > S_-) = 1$: the supports do not overlap. A random classifier gives 0.5: the distributions are identical. Everything in between is partial overlap.

This is not a metaphor. AUC-ROC is mathematically equivalent to the normalized **Mann-Whitney U statistic**, a rank-based statistic for whether one group tends to receive larger scores than the other. The U statistic counts positive-negative pairs where the positive outranks the negative, with tied pairs receiving half-credit. Divide by the total number of pairs, and you get AUC.

$$
U = \sum_{i \in \text{pos}} \sum_{j \in \text{neg}} \left(\mathbf{1}[S_i > S_j] + \tfrac{1}{2}\mathbf{1}[S_i = S_j]\right)
$$

$$
\text{AUC} = \frac{U}{n_+ \cdot n_-}
$$

So every time you compute ROC-AUC, you are computing the same rank statistic that underlies a rank-based two-sample test. If you add a null model and do inference, you can turn it into a formal test. The classifier's job, viewed this way, is to push $f_+(s)$ and $f_-(s)$ apart.

## Why this reframing matters

Once you see classification as distribution separation, several confusions dissolve.

**Why AUC is threshold-free.** You are not measuring performance at any single threshold. You are measuring a rank-based summary over all possible thresholds. Thresholds are points along the score axis — AUC summarizes the entire geometry.

**Why AUC is prevalence-insensitive.** The Mann-Whitney statistic only cares about pairwise rank comparisons between the two groups. It does not care how many samples are in each group. Whether you have 50 positives and 50 negatives or 50 positives and 5,000 negatives, the pairwise comparison logic is the same. This is why AUC-ROC does not change with class imbalance, even when imbalance is the operational problem.

**Why a random classifier gives 0.5.** If the two score distributions are identical — perfectly overlapping — then for any random positive-negative pair, the positive is equally likely to rank above or below. $P(S_+ > S_-) = 0.5$. This is just the null hypothesis of the Mann-Whitney test.

**Why AUC connects to hypothesis testing.** The Mann-Whitney U has a known null distribution. Under the null (scores are independent of labels), the AUC is 0.5. You can compute a p-value for the observed AUC against this null. In practice, people rarely do this, but the connection is real: AUC-ROC is a test statistic measuring the strength of distributional separation.

## Other distribution discrimination measures hiding in plain sight

Once this lens clicks, you start seeing it everywhere.

**KS statistic (Kolmogorov-Smirnov)**: the maximum vertical distance between the CDFs of $f_+$ and $f_-$. It measures the single threshold where separation is greatest. AUC integrates across all thresholds; KS takes the supremum.

**Cohen's d**: the difference in means divided by pooled standard deviation — assumes normal distributions but measures the same conceptual thing. How far apart are the two groups?

**Divergence measures** (KL, JS, Wasserstein): different ways to quantify how different two distributions are. These operate on the full distributional shape, not just ranks, but they live in the same conceptual family.

AUC-ROC is the rank-based, nonparametric member of this family. It asks "how separable?" without assuming any distributional form. That is both its strength (robustness) and its limitation (it ignores the shape of the tails).

## Precision-Recall: when the question shifts from separation to retrieval

AUC-PR measures something related but importantly different.

Where ROC asks "can you tell these two distributions apart?", PR asks "when you reach into the high-scoring region, what fraction of what you pull out is positive?"

Think of it this way. The score axis has a high end and a low end. The positive distribution $f_+$ and the negative distribution $f_-$ both have density across this axis. When you set a threshold and call everything above it "positive":

- **Precision** is the fraction of the density above the threshold that belongs to $f_+$
- **Recall** is the fraction of $f_+$'s total density that falls above the threshold

So PR-AUC is not measuring distributional separation in the symmetric, two-sample sense. It is measuring how cleanly the positive distribution dominates the high-scoring tail. And crucially, it depends on the **mixing proportions** — how much total mass $f_+$ and $f_-$ each contribute to the mixture.

This is why PR is prevalence-sensitive and ROC is not. ROC normalizes by each group separately. PR looks at the mixture.

## Precision@K: distribution discrimination in the top slice

Precision@K asks an even more targeted question: among the top K scored examples, how many are truly positive?

At threshold $t$, the population analogue is the positive share of the upper tail:

$$
\mathrm{Precision}(t) = \frac{\pi \cdot [1 - F_+(t)]}{\pi \cdot [1 - F_+(t)] + (1 - \pi) \cdot [1 - F_-(t)]}
$$

where $\pi$ is the positive prior. For Precision@K, $t$ is the score of the K-th ranked example.

When K is small (you only act on a handful of top predictions), you are sampling the extreme right tail of the score distribution. What matters directly is the positive-versus-negative mass above the cutoff. Locally, the density ratio $f_+(s) / f_-(s)$ near that cutoff helps explain why the tail is pure or contaminated, but the metric itself is about tail mass, not pointwise density.

This is why a model can have modest AUC-ROC (the distributions overlap substantially in the middle) and still have excellent Precision@K (the right tail is dominated by positives). AUC measures global separation. Precision@K measures local dominance.

This distinction matters operationally. In drug screening, you don't care whether the model separates inactives from actives across the full range. You care whether the top 50 predictions are enriched for actives. That is a tail-dominance question, not a distribution-separation question.

## Matthews Correlation Coefficient: the one metric that respects all four cells

MCC is the odd one in this family. It operates at a fixed threshold — it is not a curve metric — and it computes a single number from the full confusion matrix:

$$
\text{MCC} = \frac{TP \cdot TN - FP \cdot FN}{\sqrt{(TP + FP)(TP + FN)(TN + FP)(TN + FN)}}
$$

The range is $[-1, +1]$. Perfect classification gives +1, perfect inverse gives -1, and random gives 0.

What makes MCC unusual is its symmetry. It treats all four cells of the confusion matrix as equally important. Most other single-threshold metrics privilege one corner:

- **Accuracy** is dominated by the majority class
- **Precision** only looks at the predicted-positive column
- **Recall** only looks at the actual-positive row
- **F1** blends precision and recall but ignores true negatives

MCC is the Pearson correlation between the true binary labels and the predicted binary labels. That is its most revealing interpretation.

Let $Y \in \{0, 1\}$ be the true label and $\hat{Y} \in \{0, 1\}$ be the prediction. Then:

$$
\text{MCC} = \text{Corr}(Y, \hat{Y})
$$

This is the $\phi$ coefficient — a special case of Pearson correlation for two binary variables. The confusion matrix is just the 2×2 contingency table, and MCC is the correlation derived from it.

### Why MCC connects to the distribution separation view

Think of the confusion matrix at a fixed threshold as a snapshot of how well the threshold cleaves the two score distributions. MCC measures how much statistical dependence exists between the true class and the predicted class at that threshold.

If the two score distributions are perfectly separated and the threshold sits between them: $FP = 0$, $FN = 0$, and MCC = 1.

If the distributions are identical (random scores): at any threshold, $TP \cdot TN \approx FP \cdot FN$, and MCC ≈ 0.

MCC at a chosen threshold is a single-point summary of distributional separation — it tells you how much of that separation you can actually capture with a binary decision rule. If you optimize the threshold for MCC, it tells you how cleanly you can exploit the separation under that criterion. AUC tells you the separation exists. MCC tells you how cleanly a specific thresholded rule uses it.

### When MCC reveals what F1 hides

Consider a severely imbalanced problem: 950 negatives, 50 positives. A model predicts all negatives and catches zero positives:

| | Pred + | Pred - |
| --- | --- | --- |
| True + | 0 | 50 |
| True - | 0 | 950 |

- **Accuracy**: 95%. Looks excellent.
- **F1**: 0. Correctly flags the failure.
- **MCC**: conventionally reported as 0 by many libraries in this degenerate case.

Strictly speaking, the closed-form MCC expression is undefined here because the denominator is zero: the model never predicts the positive class. In practice, many implementations return 0 by convention, which matches the intuition that the classifier has no useful positive-class signal.

Now a model predicts 60 positives, catching 40 true positives:

| | Pred + | Pred - |
| --- | --- | --- |
| True + | 40 | 10 |
| True - | 20 | 930 |

- **Accuracy**: 97%
- **F1**: $\frac{2 \times 40}{2 \times 40 + 10 + 20} \approx 0.727$
- **MCC**: $\frac{40 \times 930 - 20 \times 10}{\sqrt{60 \times 50 \times 950 \times 940}} \approx 0.715$

Both F1 and MCC look reasonable. But now consider a model that predicts 200 positives, catching all 50 true positives:

| | Pred + | Pred - |
| --- | --- | --- |
| True + | 50 | 0 |
| True - | 150 | 800 |

- **Accuracy**: 85%
- **F1**: $\frac{2 \times 50}{2 \times 50 + 0 + 150} = 0.40$
- **MCC**: $\frac{50 \times 800 - 150 \times 0}{\sqrt{200 \times 50 \times 950 \times 800}} \approx 0.459$

F1 penalizes this model for the 150 false positives. So does MCC. But MCC also credits the 800 true negatives — information F1 ignores. In problems where correctly classifying negatives matters (say, a screening assay where false positives are expensive), MCC captures both sides of the tradeoff.

The general principle: **MCC is the geometric mean of the informedness and the markedness**, where informedness = TPR + TNR - 1 (Youden's J) and markedness = PPV + NPV - 1. It is a balanced view. F1 only sees the positive slice.

## The deep bridge: classification as thresholded regression

Here is where the intuition connects to something bigger.

A classifier does not begin with a hard label. What it produces — before the threshold — is a **continuous score**: a probability estimate, a logit, a margin, a distance from a hyperplane. The classification decision is what happens when you threshold that score.

$$
\hat{Y} = \mathbf{1}[f(x) > t]
$$

The score function $f(x)$ is the object you learn. The indicator function is the deployment rule. In that sense, classification is scoring plus a decision threshold.

This is not a technicality. It reorganizes the entire landscape of evaluation metrics.

### All classification metrics are functions of two distributions

The score function $f(x)$ maps each example to a point on the real line. The true positives induce one distribution of scores, $f_+(s)$. The true negatives induce another, $f_-(s)$. The metrics in this article are functionals of these two distributions (and sometimes their mixing weights):

| Metric | What it measures about $f_+$ and $f_-$ |
| -------- | ---------------------------------------- |
| AUC-ROC | Global rank separation (Mann-Whitney) |
| AUC-PR | Positive dominance in the high-score mixture |
| Precision@K | Positive share of the upper tail at the action cutoff |
| MCC | Correlation induced by a thresholded decision rule |
| KS statistic | Maximum CDF gap |
| Accuracy | Mass correctly allocated at a threshold |

They are all looking at the same two distributions from different angles.

### Regression loss optimizes distribution separation

When you train a classifier with log loss or MSE on binary targets, you are fitting $f(x)$ to move $f_+$ and $f_-$ apart. The loss function penalizes overlap. Gradient updates push positive-class scores up and negative-class scores down.

On the logit scale, log loss specifically pushes the optimal score toward the posterior log-odds of the positive class:

$$
f^*(x) = \log \frac{P(Y=1|x)}{P(Y=0|x)}
$$

By Bayes' rule,

$$
\log \frac{P(Y=1|x)}{P(Y=0|x)} = \log \frac{p(x|Y=1)}{p(x|Y=0)} + \log \frac{\pi}{1-\pi}
$$

So at the Bayes-optimal solution, the score function is the log density ratio plus a prior offset. Up to that additive constant, the thing you regress toward is the thing that measures distributional separation.

This is the circle closing. You train a regressor. It learns to separate two distributions. You evaluate it with metrics that quantify separation. You deploy by thresholding. "Classification" is just the user-facing wrapper around distribution discrimination.

### What the PR-style counterpart looks like in regression

There is no single canonical regression analogue of PR-AUC or Precision@K, because regression has no intrinsic positive class and therefore no native decomposition into $TP$, $FP$, and predicted-positive purity. But if the operational goal is upper-tail discovery, the analogue appears naturally once you reinterpret the problem as ranking over the response variable. Let $\hat{y}(x)$ be the model score and $Y$ the continuous target. Then the tail-focused question is not "how small is $(Y-\hat{y})^2$ on average?" but rather "among the points with largest $\hat{y}$, how much of the high-response region have we captured?"

That produces two related families of metrics. One family is global ranking quality: Spearman's $\rho$, Kendall's $\tau$, and the concordance index ask whether larger predicted scores tend to align with larger realized outcomes across the full range. The other family is tail-focused capture: top-$K$ mean observed response, lift or cumulative gain, or, when the outcome is nonnegative and additive, the fraction of total outcome captured above a score threshold. Those are closer in spirit to Precision@K because they care about what happens in the extreme right tail rather than average ordering everywhere.

If you want the closest PR-style construction, define a tail event $Z_u = \mathbf{1}[Y > u]$ for some high quantile threshold $u$, and then evaluate PR-AUC or Precision@K on $(\hat{y}(x), Z_u)$. In other words, PR-style metrics do not disappear in regression; they re-emerge when a continuous response is converted into a rare upper-tail event. The underlying split is the same as in classification: global metrics like RMSE evaluate fidelity across the full outcome distribution, while tail-focused ranking or capture metrics evaluate how well the model concentrates high-response cases into the right tail of the score distribution.

### Why this reframing clarifies model improvement

When a model has high AUC but poor Precision@K, the distributions are globally separated but their right tails still overlap. The fix is not "classification tricks" — it is improving the score function in the high-scoring region. That is a regression problem: make $f(x)$ larger for true positives and smaller for true negatives, specifically in the region where the model is most confident.

When a model has high AUC-ROC but poor AUC-PR, the global separation is good but the mixing proportions are destroying precision. The class prior $\pi$ is so small that even modest negative-class density in the high-scoring region overwhelms the positive density. The fix may be recalibration, cost-sensitive reweighting, or threshold adjustment — none of which are "classification" changes. They are adjustments to how you interpret the regression output.

When MCC is low despite reasonable accuracy, the threshold is not carving the score axis at the right point, or the score function is not producing a clean separation. Again — debug the regression, not the classification.

### How you train for Precision@K

Once you see Precision@K as a right-tail problem, the training objective becomes clearer. You are trying to reduce negative-class mass in the high-scoring region. In the density-ratio language, you want the upper tail to be a region where $f_+(s)$ dominates $f_-(s)$.

Standard cross-entropy does not target this directly. It optimizes average log-likelihood over the full dataset. That is often a good general-purpose objective, but it does not distinguish sharply enough between easy negatives scoring 0.01 and dangerous negatives scoring 0.85. Only the latter actually poison the top of the ranking.

In practice, the most useful approaches are variations on the same idea: focus learning pressure on the negatives that leak into the right tail.

1. **Hard negative mining.** Train disproportionately on negatives that already score high or look deceptively similar to positives. Online hard example mining, triplet objectives with hard negatives, and contrastive setups all do this in different ways. You are explicitly telling the model: these are the $f_-$ samples contaminating the tail; push them down.
2. **Focal loss.** Focal loss downweights easy examples by a factor of $(1-p_t)^\gamma$, where $p_t$ is the model-assigned probability of the correct class. The effect is that optimization capacity shifts toward ambiguous and misranked cases, which are exactly the examples living near the overlap region that hurts Precision@K.
3. **Learning-to-rank losses.** Pairwise and listwise ranking objectives such as LambdaRank, LambdaMART, and approximate NDCG surrogates target the ordering itself rather than calibrated probabilities. They penalize swaps where a negative outranks a positive, often with extra weight on top-of-list mistakes. That makes them especially natural when the operational question is literally about the top $K$.
4. **Two-stage cascades.** A common retrieval pattern is candidate generation followed by re-ranking. The first model is optimized for broad recall and global separation. The second model only sees the shortlist and uses a more precision-focused objective to clean the right tail. The first stage finds the region; the second stage purifies it.
5. **Cost-sensitive weighting.** Increasing the effective cost of false positives changes both the fitted score function and the threshold you prefer at deployment time. This is less targeted than hard-negative or ranking-based methods, but it is often simple and effective when false positives are the expensive failure mode.

The non-obvious point is that exact Precision@K is not a smooth objective. It changes in discrete jumps when examples swap rank positions. That means standard gradient methods usually optimize a surrogate or a smooth relaxation instead of the exact metric itself. The art is choosing the surrogate that matches your failure mode: hard-negative methods when tail contamination is localized, focal loss when easy examples dominate the gradient, ranking losses when order matters more than calibration, and cascades when you can afford to separate recall from precision.

## The unifying picture

$$
\text{Raw features} \xrightarrow{f(x)} \text{Score (regression)} \xrightarrow{\text{threshold}} \text{Class label}
$$

Every evaluation metric looks at a different aspect of the middle step — the quality of the score function as a separator of two distributions.

- **AUC-ROC**: global rank separation, prevalence-independent
- **AUC-PR**: positive-class recovery in the mixture, prevalence-dependent
- **Precision@K**: tail purity, local density ratio
- **MCC**: balanced correlation at a fixed threshold, uses all four confusion matrix cells
- **F1**: positive-focused tradeoff at a fixed threshold, ignores true negatives

No single metric is the "right" one. They each project the same underlying geometry onto a different axis. The deep similarity you feel between ROC-AUC and distribution discrimination is not an analogy — it is an identity. And once you hold that identity, every other metric becomes a variation on the same theme: how well does the learned score function push two distributions apart, and how does that separation look from the angle the user actually cares about?

If the concrete decision in front of you is specifically ROC versus PR under rare positives, the companion piece [AUC-ROC vs AUC-PR](/posts/auc-roc-vs-auc-pr-what-changes-when-positives-are-rare/) is the operational view of the same geometry.

References: [Mason and Graham, 2002](https://doi.org/10.1023/A:1016338720772) (AUC and Mann-Whitney equivalence); [Chicco and Jurman, 2020](https://doi.org/10.1186/s12864-019-6413-7) (MCC advantages over F1); [Hand, 2009](https://doi.org/10.1007/s10994-009-5119-5) (coherent evaluation measures for binary classifiers).
