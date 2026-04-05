---
title: "Beyond Correlation II: What Foundation Models Actually Learn — and What the Eigenspectrum Reveals"
date: 2026-04-06
categories: [Machine Learning]
tags: [foundation-models, model-evaluation, eigenspectrum, single-cell, computational-biology]
math: true
---

*This is Part II of the **Beyond Correlation** series: [Part I](/posts/beyond-correlation-i-mahalanobis-pca-geometry/) | Part II (this article) | [Part III](/posts/beyond-correlation-iii-optimal-transport-cell-state/)*

---

The vision behind single-cell foundation models is compelling. Train a transformer on tens of millions of cells — spanning tissues, developmental stages, disease states — and the model learns a general representation of cellular biology. scGPT, Geneformer, scFoundation, scBERT, UCE: each has been trained on atlas-scale data with the ambition of building a "virtual cell" that can predict how the transcriptome responds to perturbation.

The standard evaluation plots measured expression against predicted expression across all genes, computes Pearson's R, and reports a value around 0.95. That number is real. But working through what it actually measures — via the same spectral framework from Part I — reveals that most of the R is determined by the structure of the transcriptome itself, before any model contributes anything specific about regulation. And two independent benchmarking studies — Ahlmann-Eltze, Huber, and Anders (Nature Methods, 2025) and Csendes et al. (BMC Genomics, 2025) — confirm empirically what the decomposition predicts: on the task that matters most — predicting transcriptomic responses to genetic perturbation — these models do not yet outperform deliberately simple baselines.

Understanding why connects the eigenspectrum geometry from Part I to a concrete, consequential question in machine learning for biology.

## Decomposing the 0.95

Start with what gene expression looks like across a transcriptome. Ribosomal proteins, cytoskeletal components, and mitochondrial enzymes are expressed at high levels in virtually every cell type. Thousands of genes sit at moderate levels through constitutive programs. Thousands more are barely expressed or silent: olfactory receptors in a liver sample, developmental regulators for stages long past, transcription factors for lineages the cell does not belong to.

This creates an enormous dynamic range. In log₂TPM space, values span roughly 0 to 15. The expression level of any given gene is determined overwhelmingly by what that gene is and what cell type you are measuring — not by the specific condition or perturbation in your experiment.

A simple decomposition makes this precise. Write the measured expression of gene i as two components:

xᵢ = μᵢ + δᵢ

where μᵢ is the baseline expression level — determined by gene identity, cell type, and constitutive programs — and δᵢ is the condition-specific component: the part that changes across perturbations, time points, or biological contexts. The model's prediction is:

yᵢ = μ̂ᵢ + εᵢ

where μ̂ᵢ is the model's estimate of the baseline and εᵢ is its prediction error.

The Pearson correlation between measured and predicted, across all p genes:

R(x, y) = cov(x, y) / (σₓ · σᵧ)

Expanding the covariance — and treating μ and δ as approximately separable across genes, since baseline expression levels and condition-specific regulation are largely driven by different genomic features (this is an approximation; in practice the degree of separability depends on the dataset and normalization):

cov(x, y) = cov(μ + δ, μ̂ + ε) = cov(μ, μ̂) + cov(δ, ε) + cross terms

The dominant term is cov(μ, μ̂) — agreement on baseline expression. The variance of μ across genes is large (order of magnitude: 10–20 in squared log₂ units), driven by promoter architecture, gene length, codon usage, and chromatin accessibility at housekeeping loci. The variance of δ — the condition-specific component — is typically much smaller (order of magnitude: 0.5–2.0 for most genes, though this varies with normalization, gene filtering, and the strength of the perturbation).

A back-of-the-envelope calculation illustrates the implication. When the model captures the baseline well (cov(μ, μ̂) ≈ var(μ)):

R ≈ √(var(μ) / (var(μ) + var(δ)))

If var(μ) ≈ 15 and var(δ) ≈ 1.5 — representative orders of magnitude, not universal constants — this gives R ≈ 0.95. The precise numbers are dataset-dependent, but the qualitative point is robust: any model that captures mean expression levels well will achieve a high across-gene R, because the variance ratio strongly favors the baseline component. Multiple benchmarks confirm this empirically: in raw expression space, all models (including trivial ones) achieve Pearson above 0.95, and the number becomes uninformative.

A model that perfectly memorizes mean expression for each gene — with zero knowledge of regulatory dynamics — achieves approximately this R on any held-out sample from the same cell type. Silent genes cluster near zero in both measured and predicted. Housekeeping genes cluster near the top. The correlation follows from the transcriptome's architecture.

## Twenty Thousand Genes, Far Fewer Independent Facts

There is a second structural issue, connecting directly to the gene-gene covariance matrix from Part I.

When we compute R across 20,000 genes, we implicitly treat each gene as an independent observation about prediction accuracy. But genes are organized into co-regulated modules: genes controlled by the same transcription factors, genes in the same pathway, genes responding to the same upstream signal. The covariance matrix Σ encodes this structure.

If 500 genes belong to the ribosomal protein program and the model captures that program through a single learned feature — a TOP motif in the 5' UTR, or a shared promoter architecture — those 500 genes do not provide 500 independent pieces of evidence about model quality. They provide approximately one piece of evidence, repeated 500 times. The scatter plot registers 500 concordant points, but it is one biological fact echoed across a co-regulated module.

The eigendecomposition of Σ = UΛUᵀ makes this quantitative. In single-cell data from one cell type, the eigenvalue spectrum typically decays steeply: a small number of principal components capture the majority of total variance, corresponding to major co-regulated programs (ribosomal biogenesis, mitochondrial metabolism, cell cycle, stress response). The exact decay rate depends on cell type heterogeneity, normalization, and gene filtering — but the qualitative pattern is consistent. The effective number of independent observations is bounded by the effective rank:

n_eff = (tr(Σ))² / tr(Σ²) = (Σᵢ λᵢ)² / (Σᵢ λᵢ²)

When a few eigenvalues dominate, this ratio is far smaller than p. Across datasets, n_eff is typically orders of magnitude below the nominal gene count — meaning the 20,000 points on a scatter plot contain far fewer independent facts than they appear to.

This is a general inflation mechanism that arises whenever correlated observations are treated as independent evidence. The gene-gene covariance creates correlated echoes of a single learned feature across co-regulated genes. The nominal count of supporting observations overstates the independent evidence by a factor proportional to the module size.

## Where R Looks and Where It Does Not

The geometric interpretation from Part I sharpens this further.

Pearson's R, computed across genes, is a variance-weighted inner product. The genes that contribute most are those at the extremes of the expression distribution — the most highly expressed and the most silent. These sit along the top principal components of the transcriptome.

In PCA coordinates z = Uᵀ(x − μ⃗), the contribution of the k-th principal component to R scales with its eigenvalue λₖ. The whole-transcriptome R is dominated by the components with the largest λₖ — the directions corresponding to constitutive programs, housekeeping identity, and cell type signature. Prediction errors in the lower-variance PCs — where condition-specific regulation, subtle enhancer effects, and context-dependent variation reside — contribute negligibly because their eigenvalues are small relative to the total.

This is the geometric dual of the Mahalanobis framework from Part I:

- **Mahalanobis**: weight proportional to 1/λₖ (sensitive to deviations in tightly constrained directions)
- **Pearson R**: weight proportional to λₖ (sensitive to agreement along high-variance directions)

The practical consequence: the signal that distinguishes a model with genuine predictive insight from one that has memorized the baseline lives in the lower-variance principal components — exactly where whole-transcriptome R has the least sensitivity.

## What Happens When You Test on Perturbations

The variance decomposition makes a specific, testable prediction: a foundation model trained on observational single-cell data should excel at capturing the baseline μ — the constitutive expression landscape — and struggle with the condition-specific component δ, particularly for perturbations outside its training distribution.

Two independent benchmarking studies tested exactly this. Ahlmann-Eltze, Huber, and Anders (Nature Methods, 2025) benchmarked five foundation models (scGPT, scFoundation, and others) plus additional deep learning architectures against deliberately simple baselines for predicting transcriptomic responses to genetic perturbation. Csendes et al. (BMC Genomics, 2025) conducted a parallel evaluation with overlapping conclusions. The baselines included a "no change" model (predict the control expression — literally, predict μ), an additive model (sum individual log fold-changes), and a linear model with learned embeddings.

The result: none of the foundation models consistently outperformed the baselines. On single unseen perturbations, the simple approaches matched or exceeded the deep learning models. Some foundation model predictions "did not vary across perturbations" — meaning the model returned approximately the same transcriptome regardless of which gene was perturbed. It was, in the language of the variance decomposition, predicting μ and ignoring δ entirely. Csendes et al. showed that even a "train mean" baseline — predicting the average expression profile — can outperform foundation models when evaluated in differential expression space.

The linear model with perturbation-specific embeddings — trained on related perturbation data, not atlas-scale observational data — occasionally matched the deep learning models. The key variable was not model complexity or training set size, but the type of training data: models pretrained on perturbation measurements had access to information about δ that observational pretraining cannot provide.

This is not a failure of engineering. It is a structural property of what observational data contains.

An important nuance: the same benchmarks suggest that foundation models may still be valuable as representation learners, even when their decoders fail as perturbation simulators. Ahlmann-Eltze et al. found that plugging pretrained embeddings into a simple linear model can perform comparably to the model's own prediction head. The representation captures something useful about transcriptomic structure — the model has learned a compressed version of Σ that transfers across contexts. The limitation is specific: these representations encode the observational landscape well but do not, on their own, extrapolate to interventional outcomes.

## The Structural Boundary: Covariance Is Not Causation

The deepest insight from this series crystallizes here.

Observational single-cell data, no matter how abundant, estimates the gene-gene covariance Σ. This is the correlational structure: which genes tend to go up and down together across naturally occurring cellular states. With enough cells, Σ can be estimated with arbitrary precision. Foundation models trained on millions of cells learn this structure well — they capture co-expression modules, cell type signatures, developmental trajectories, and the full landscape of natural transcriptomic variation.

But predicting what happens when you intervene — when you silence a gene and the network propagates the effect — requires a different object entirely. Call it **N**: the network propagation matrix, where entry Nᵢⱼ represents how much gene i changes when gene j is directly perturbed. The observed differential expression after a perturbation is:

δy = Nγ + noise

where γ is the vector of direct perturbation effects.

Σ and N are related but fundamentally distinct. In causal inference terms, Σ encodes P(xᵢ | xⱼ) — the conditional distribution under observation. N encodes P(xᵢ | do(xⱼ)) — the distribution under intervention. The do-operator changes the generative process: instead of observing the system, you are forcing a variable to a value and watching the network respond.

Σ is symmetric by construction. N is not. If gene A regulates gene B, observing high A predicts high B (Σ captures this). But intervening on B does not change A, because the arrow points from A to B. The observational correlation is symmetric; the interventional effect is directional.

In graphical model theory, this is Markov equivalence: multiple distinct network topologies can produce the same Σ. The covariance matrix is a projection of the full causal structure onto pairwise observational statistics. Information is lost in that projection. A model that has learned Σ perfectly — that generates realistic single-cell profiles, interpolates between cell states, identifies co-expression modules with high fidelity — has learned the observational distribution. It has not learned N.

This is why scaling observational data has diminishing returns for perturbation prediction specifically. Going from one million to ten million to one hundred million cells refines Σ but does not bring the model closer to N. The gap is structural — it is not closed by more data of the same type, but by data of a different type: perturbation experiments that directly reveal columns of N.

The experiments that do provide information about N — Perturb-seq, CROP-seq, and related pooled CRISPR screens — directly observe the response to intervention. Each perturbation reveals approximately one column of N: the transcriptomic effect of perturbing one gene. Building up N column by column requires perturbing genes individually (or in combinatorial designs) and measuring the response. This is expensive, but it is the right kind of data, because it samples the interventional distribution.

A caveat: even perturbation data does not fully identify N without additional assumptions about the model class — linearity, absence of hidden confounders, no feedback loops, time-scale separation. The observed response to knocking out gene j is the total network effect, including indirect paths and compensatory regulation, not the direct edge weight. Recovering the sparse direct-effect structure from total perturbation responses is itself an inverse problem. The point is not that perturbation data gives N for free, but that it provides the right type of information — interventional rather than observational — from which N can in principle be estimated under stated assumptions.

These benchmark results follow naturally from this framework. The linear model with perturbation embeddings learned from related perturbation data performed well because it had access, however indirectly, to columns of N. The foundation models trained on atlas-scale observational data had learned Σ to high precision — enough to generate realistic expression profiles and achieve R ≈ 0.95 on held-out observations — but Σ does not contain the directional, causal information needed to predict perturbation outcomes.

## Complementary Metrics for Model Evaluation

The variance decomposition suggests a family of metrics that separate baseline memorization from regulatory prediction. These are not replacements for whole-transcriptome R — they are additional lenses, each isolating a different component of performance.

**Fold-change correlation.** For paired conditions (treatment vs. control), compute measured log₂FC and predicted log₂FC for each gene. The correlation between these vectors eliminates μ entirely — the baseline cancels in the subtraction — and isolates the model's ability to predict δ. This is the most direct test of whether a model has learned regulatory dynamics rather than gene identity.

**Per-gene correlation across conditions.** For each gene individually, compute the correlation between measured and predicted expression across a panel of conditions. This inverts the axis: instead of "across genes, does the model track expression levels?" it asks "for gene X, does the model correctly predict when it goes up and when it goes down?" A model with high across-gene R but low average per-gene R has captured var(μ) but not var(δ).

**Expression-stratified R.** Partition genes into quantile bins by mean expression level and compute R within each bin. This removes the between-bin variance — the primary driver of whole-transcriptome R — and reveals per-bin accuracy. The informative signal is typically in the middle and lower bins, where regulatory variation dominates over baseline identity.

**Whitened-space R.** Project both measured and predicted expression into PCA space, divide each component by √(λₖ), and compute R. This equalizes the weight of each independent direction in the eigenspectrum. In this space, the baseline agreement on housekeeping gene levels receives no more weight than the model's ability to predict subtle regulatory variation. Computationally straightforward given an SVD of the expression matrix.

**Perturbation-specific δ metrics.** For perturbation prediction specifically, L₂ distance on selected differentially expressed genes, or the Pearson δ measure on fold-changes (as used in recent benchmarks), directly evaluates the model's knowledge of N rather than Σ. These metrics are harder to inflate with baseline memorization.

Each of these controls for a different source of the baseline inflation. Used together, they decompose the R = 0.95 into its constitutive components — how much comes from the eigenspectrum of μ, how much from co-expression module redundancy, and how much from genuine condition-specific prediction.

## The Hierarchy of Data

The full series reveals a hierarchy in what different types of data provide:

**Observation at scale → Σ** (symmetric, correlational). Single-cell atlases, bulk RNA-seq compendia, and multi-tissue panels refine the gene-gene covariance with increasing precision. Foundation models trained on this data learn the observational distribution — the landscape of natural variation, cell type boundaries, and co-regulated programs. This is valuable. It is necessary for many downstream tasks.

**Intervention → N** (asymmetric, causal). Perturb-seq, CROP-seq, and systematic knockdown/knockout screens directly sample the interventional distribution. Each experiment reveals a column of N — the causal consequence of removing one gene from the network. This information is qualitatively different from Σ and cannot be derived from observational scaling alone.

**Inverse problem → γ** (the specific decomposition). Given N and a perturbation measurement δy, recovering the sparse vector of direct effects γ requires solving δy = Nγ + noise. This inverse problem is ill-posed when columns of N overlap (network modularity), and it requires additional constraints — multiple reagents, orthogonal modalities, temporal measurements — to narrow the solution space. Breaking the degeneracy requires observing the same signal through different correlation structures.

Each level requires qualitatively different data, not just more of the same.

## The Series So Far

Looking back across the first two parts, the mathematical structure repeats.

**Part I.** The object is the gene-gene covariance Σ. Correlation treats genes as independent axes; Mahalanobis distance reweights by the eigenstructure, amplifying deviations along constrained directions.

**Part II (here).** Two objects, one framework. First, the eigenspectrum of Σ reveals why whole-transcriptome R is dominated by baseline expression rather than regulatory prediction — the same inflation mechanism that arises whenever correlated features are treated as independent. Second, the distinction between Σ and N — between observational covariance and causal network propagation — explains why foundation models trained on atlas-scale data achieve high R on held-out observations but fail to predict perturbation outcomes. The resolution requires both complementary evaluation metrics (to diagnose what R actually captures) and complementary data types (to provide the interventional information that observational training cannot).

The unifying principle across both parts: a correlated observation space inflates a summary statistic that implicitly assumes independence. The resolution requires decomposing the summary into its spectral components — either by reweighting (Mahalanobis) or by variance stratification and causal data integration (the approach described here).

## The Mental Models

**The R = 0.95 is a property of the transcriptome's eigenspectrum, not of the model.** The variance ratio var(μ)/var(δ) determines the floor. A mean-expression lookup table achieves it. Foundation models report it. The number reflects transcriptome architecture — the dynamic range between silent and housekeeping genes — more than it reflects predictive capability.

**Twenty thousand genes provide far fewer independent observations than the gene count implies.** Co-regulated modules echo the same learned feature across hundreds of genes. The effective sample size for evaluating model quality is the effective rank of Σ, not the gene count. Scatter plots with 20,000 points can be visually convincing while containing fewer independent facts than a modest sample survey.

**Pearson R and Mahalanobis distance weight the eigenspectrum in opposite directions.** R is dominated by the top principal components (high-variance, constitutive). Mahalanobis distance is sensitive to the bottom components (low-variance, tightly regulated). The signal that separates genuine regulatory prediction from baseline memorization lives in the lower eigenvalues — where R has the least sensitivity.

**Σ is necessary but not sufficient for perturbation prediction.** Observational data at any scale estimates Σ — the correlational landscape. Perturbation prediction requires N — the causal propagation structure. Σ is symmetric; N is not. Multiple network topologies produce the same Σ. The gap is structural: it is not closed by more observations, but by interventional data that reveals the directional architecture of the network.

**Decomposing R separates what models learn from what they memorize.** Fold-change correlation, per-gene correlation, expression-stratified R, and whitened-space R each control for a different source of baseline inflation. Together, they provide a spectral decomposition of model performance that isolates the components carrying genuine biological insight.

The linear algebra across this series is standard. What I find useful is recognizing that the same eigenvalue-weighted geometry — the covariance matrix as central object, its spectral decomposition as diagnostic tool, and the implicit independence assumption as the source of inflation — connects distance metrics, model evaluation, and causal inference as instances of the same underlying structure.

But everything so far has compared individual points — single expression profiles, single model predictions. In single-cell biology, a condition is not one point. It is a cloud of thousands of cells, each at a different position in transcriptomic space. Comparing two conditions means comparing two distributions, and the standard metrics break down in new ways when you try to measure the distance between clouds rather than between points. Part III will take the geometric framework from this series and extend it from points to distributions — using optimal transport to ask not "how different are these density functions?" but "how much does it cost to physically reshape one cell population into another?"

---

*Previous: [Part I — The Distance Your PCA Plot Actually Measures](/posts/beyond-correlation-i-mahalanobis-pca-geometry/) | Next: [Part III — Optimal Transport and the Geometry of Cell State](/posts/beyond-correlation-iii-optimal-transport-cell-state/)*
