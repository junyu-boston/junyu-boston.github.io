---
title: "Beyond Correlation I: The Distance Your PCA Plot Actually Measures"
date: 2026-04-05
categories: [Mathematics]
tags: [linear-algebra, pca, mahalanobis-distance, covariance, computational-biology]
math: true
---

*This is Part I of the **Beyond Correlation** series: Part I (this article) | [Part II](/posts/beyond-correlation-ii-foundation-model-eigenspectrum/) | [Part III](/posts/beyond-correlation-iii-optimal-transport-cell-state/)*

---

In computational biology, correlation is the first tool we reach for. Sample correlation heatmaps, hierarchical clustering, PCA plots — these are the universal starting points for any high-dimensional dataset. RNA-seq, proteomics, metabolomics: compute pairwise correlations, cluster the samples, check for outliers. It works, it scales, and it has carried the field remarkably far.

But correlation between samples is a specific geometric choice, and it carries a hidden assumption. Understanding what that assumption is — and what opens up when you move past it — reveals a deeper layer of structure inside the PCA plots we already use every day.

## What Sample Correlation Actually Computes

When we correlate two samples across p genes, we compute a normalized dot product of their expression vectors. Geometrically, this is a measure of the angle between two points in gene space. Pearson correlation treats every gene as an equally weighted, independent axis. The contours of equal similarity are spherical — the same geometry as Euclidean distance, just rescaled.

This is a perfectly valid measure of profile similarity. Two samples with high correlation have expression vectors pointing in similar directions. Two with low correlation point differently. It answers a clean question: "How similar are these profiles, gene by gene?"

But notice the assumption baked in: every gene contributes equally. A difference in a gene that fluctuates wildly across all conditions counts the same as a difference in a gene that is normally rock-stable. The correlation coefficient has no way to distinguish the two, because it has no model of how genes relate to each other.

## From Samples to Genes: Where Covariance Enters

The moment you ask "which differences between samples are actually surprising?" you need something that sample correlation does not provide: the gene-gene covariance structure.

Consider a concrete example. Two treatment samples differ from controls on Gene A (high natural variance, fluctuates in every experiment) and Gene B (extremely stable, tightly regulated across conditions). Sample correlation weights both shifts equally. But biologically, the shift in Gene B is far more informative — it broke a pattern that normally holds. The shift in Gene A might just be noise.

To distinguish "expected variation" from "surprising deviation," you need to know how genes co-vary — which genes move together, which are tightly constrained, and how the axes of natural variation are oriented. That information lives in the p × p gene-gene covariance matrix, Σ.

This is the dimension jump from univariate to genuinely multivariate thinking. The fundamental object changes from a scalar variance to a covariance matrix, and the shape of uncertainty changes from an interval to a tilted ellipsoid. Variables that are correlated produce an ellipsoid whose axes are rotated relative to the original gene coordinates. The box you get from treating genes independently — the Cartesian product of individual confidence intervals — includes impossible corners and misses plausible regions along the correlated directions.

It is not "more of the same." It is a different geometry.

## The Covariance-Aware Ruler: Mahalanobis Distance

Once you accept that the gene-gene covariance defines the shape of your data, a natural question follows: how do you measure how far a sample is from the center of that shape?

Mahalanobis distance is the answer. For a point x with population mean μ and covariance Σ:

d²(x, μ) = (x − μ)ᵀ Σ⁻¹ (x − μ)

Where sample correlation asks, "How similar are these expression profiles?" — treating all genes as interchangeable coordinates — Mahalanobis distance asks, "How many standard deviations away is this sample, after accounting for how genes co-vary?"

It replaces the sphere with an ellipsoid that matches the data's natural shape. Directions where the data varies a lot count less. Directions where it is tightly constrained count more. Correlations rotate the axes. A difference along a noisy, high-variance gene direction is downweighted — "that gene fluctuates anyway." A difference along a stable, low-variance direction is amplified — "that gene is normally locked, so this shift is meaningful."

Correlation tells you how similar two profiles look. Mahalanobis distance tells you how unusually different they are, given what the data's own structure says to expect. Both are useful. They answer different questions.

## PCA Makes the Connection Visible

Here is where PCA, correlation, and Mahalanobis distance reveal themselves as layers of the same geometric framework.

The eigendecomposition of the covariance matrix gives you:

Σ = U Λ Uᵀ

where U contains the principal directions (eigenvectors) and Λ = diag(λ₁, ..., λₚ) holds the variance along each principal component.

If you express your point in PCA coordinates — z = Uᵀ(x − μ) — the Mahalanobis distance simplifies to:

d² = z₁²/λ₁ + z₂²/λ₂ + ... + zₚ²/λₚ

Take each PC coordinate, square it, divide by the variance along that PC, and sum.

There is an even more elegant way to see this. Define whitened coordinates yᵢ = zᵢ / √λᵢ, and Mahalanobis distance becomes:

d² = ‖y‖²

Ordinary Euclidean distance — but measured after PCA rotation and variance normalization. PCA gives you the axes. Whitening gives you the scale. Mahalanobis distance is simply Euclidean distance in the whitened PCA space.

This is the progression: sample correlation uses raw gene coordinates (spherical geometry). PCA rotates into the natural axes of variation. Mahalanobis distance completes the picture by rescaling those axes according to how much variance each one carries. Each step builds on the last.

Every time you eyeball a PCA plot and mentally judge "that sample looks like an outlier," you are performing an approximate Mahalanobis calculation in your head. The math just makes it precise.

## The High-Dimensional Reality

This framework is clean in theory. In practice, there is a constraint that matters deeply for genomics.

Mahalanobis distance requires the inverse of the covariance matrix. When you estimate that matrix from data, you need substantially more samples than variables for the estimate to be stable. Every eigenvalue appears in the denominator. If you have 20,000 genes and 30 samples, the smallest eigenvalues are essentially noise — tiny numbers estimated from far too little data. Dividing by noise amplifies noise.

The paradox is sharp: Mahalanobis distance places the most trust in the lowest-variance principal components, because deviations there are the most "unexpected." But those are exactly the components whose eigenvalues are estimated the worst when the sample size is small relative to the number of variables.

If n < p, the sample covariance matrix becomes rank-deficient — non-invertible. There is a standard workaround: eigendecompose the n × n Gram matrix XXᵀ instead, extract the same non-zero eigenvalues via SVD duality, and compute M-distance in the reduced subspace. This solves the computational singularity. It does not solve the statistical one.

With 30 samples, you recover at most 29 non-zero eigenvalues. The true population covariance has up to 20,000. Your M-distance captures variation only in the 29 directions your samples span — any signal living in the remaining dimensions is invisible. And even within those 29 directions, the tail eigenvalues are noise-dominated.

This is exactly why sample correlation remains so widely used in genomics, and rightly so — it requires no covariance inversion and is stable regardless of the n/p ratio. The question is not which metric is "better" in the abstract. It is which questions each one can reliably answer given the data you actually have. When n is small relative to p, sample correlation is robust and interpretable. When n is large enough to support covariance estimation — or when you apply regularization, shrinkage, or reduced-rank models — Mahalanobis distance unlocks a layer of sensitivity that correlation alone cannot access.

## The Inverse Problem Behind Every Distance Score

There is a deeper conceptual point that connects the geometry above to the biology we actually care about.

A high-dimensional transcriptomic response — say, the expression profile of a cell after a perturbation — can be modeled as a mixture of underlying biological programs:

y = Aβ + ε

where A is a dictionary of latent gene programs, β is their activation weights, and ε is noise.

Recovering β from y is an inverse problem. And when the columns of A are correlated (which they always are, because biological pathways share genes), the problem is ill-posed. Multiple combinations of program activations can produce the same observed expression pattern. This is a linear algebra statement: the system is underdetermined or near-singular.

Any scalar distance score — whether sample correlation, Euclidean distance, or Mahalanobis distance — tells you how unusual an observation is under a model. It does not tell you why it is unusual. A large distance could reflect one strongly activated pathway, or many weakly activated ones, or a single misregulated gene sitting in a low-variance direction. The distance is the same; the biology is entirely different.

This is especially relevant in RNA-seq, where measured expression is an endpoint that blends direct perturbation effects with downstream cascading responses — network ripple effects, feedback loops, compensatory mechanisms. Decomposing that endpoint into "causes" requires more than geometry. It requires additional constraints: time-series data, perturbation experiments, prior network knowledge, or regularization assumptions that encode biological structure.

Geometry summarizes patterns. Causal identification requires constraints. These are complementary tools, not substitutes.

## The Mental Models That Transfer

Working through the linear algebra of correlation, PCA, and Mahalanobis distance has given me a set of mental models that I apply constantly in computational biology:

**Correlation, PCA, and Mahalanobis distance are layers, not alternatives.** Correlation uses raw gene coordinates. PCA rotates into natural axes of variation. Mahalanobis distance rescales by variance. Each builds on the previous one, and each answers a progressively more refined question about what is "unusual" in your data.

**Gene-gene covariance is what separates "different" from "surprising."** Sample correlation measures profile similarity. Mahalanobis distance measures how unexpected a difference is, given the covariance structure. When genes are co-regulated in modules and pathways share upstream signals, that covariance structure carries real biological information.

**The n/p ratio determines which layer is trustworthy.** When samples are scarce relative to genes, sample correlation is robust and covariance inversion is fragile. When samples are abundant or regularization is applied, the covariance-aware layer becomes accessible. Knowing where you sit on this spectrum is a practical decision that shapes every downstream analysis.

**A scalar distance is not a causal decomposition.** Distance tells you "how far from typical." It does not tell you "how many things went wrong" or "what pathway is responsible." Only additional experimental design and biological constraints can bridge that gap.

These are not abstract principles. They are operational decisions that determine which analyses I trust, which methods I recommend for a given sample size, and how I communicate results to biologists making real decisions about therapeutic programs.

The geometry inside that PCA plot runs deeper than the visualization suggests. Correlation is where we start. It does not have to be where we stop.

---

*Next in the series: [Part II — What Foundation Models Actually Learn](/posts/beyond-correlation-ii-foundation-model-eigenspectrum/)*
