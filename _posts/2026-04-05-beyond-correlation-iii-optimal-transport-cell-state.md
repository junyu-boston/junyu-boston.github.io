---
title: "Beyond Correlation III: Optimal Transport and the Geometry of Cell State"
date: 2026-04-05
categories: [Mathematics]
tags: [optimal-transport, wasserstein-distance, manifold-geometry, single-cell, computational-biology]
math: true
---

*This is Part III of the **Beyond Correlation** series: [Part I](/posts/beyond-correlation-i-mahalanobis-pca-geometry/) | [Part II](/posts/beyond-correlation-ii-foundation-model-eigenspectrum/) | Part III (this article)*

---

The first two parts of this series kept coming back to the same problem: the standard metrics we use in computational biology quietly assume that genes are independent and that the space is flat. Correlation inflates when eigenvalues are unequal. Whole-transcriptome R captures baseline expression, not regulatory signal. In each case, the fix was the same — take the geometry of the data seriously.

This final article changes the question. Parts I and II compared individual expression profiles — single points in gene space. But in single-cell biology, a condition is not one point. It is a cloud of thousands of cells, each sitting at a slightly different position in transcriptomic space. Comparing treated versus untreated now means comparing two clouds, and the choice of how to compare them carries the same hidden geometric assumptions we have been unpacking throughout this series.

The metric that handles this correctly is the Wasserstein distance, from optimal transport theory. The core idea is simple: instead of asking "how different are these two density functions?", it asks "how much work would it take to physically reshape one cloud of cells into the other?" That shift — from comparing densities to measuring the cost of movement — turns out to be exactly what single-cell biology needs.

## Why the Obvious Approaches Break Down

Start with the most natural instinct. You have two single-cell experiments — treated and untreated — each giving you a cloud of cells in gene expression space. You want a single number that says how different these two conditions are.

**Approach 1: Compare the averages.** Compute the mean expression vector for each condition and measure the Euclidean distance between them:

d = ‖μ_treated − μ_untreated‖₂

This reduces each cloud to a single point, then compares two points. The problem is obvious once you think about it: two clouds can have the exact same mean but completely different shapes. Imagine a unimodal cluster versus a bimodal one with two subpopulations sitting on opposite sides of the centroid. The means are identical. The distance is zero. The biology is completely different.

**Approach 2: Compare the density functions.** Use a statistical divergence — KL divergence, Jensen-Shannon, or total variation distance — to compare the probability density of each cloud. These are more sophisticated, but they share a fatal property: they compare the densities *at each point in space independently*, without knowing how close or far apart those points are.

Here is what that means concretely. Suppose all treated cells sit at expression state A, and all untreated cells sit at expression state B. The KL divergence is infinite — the two distributions have no overlap, so the ratio P(x)/Q(x) blows up. This is the same answer whether A and B are neighboring cell states separated by a tiny expression shift, or completely unrelated cell types on opposite sides of the transcriptome. Total variation says both cases are "maximally different" (TV = 1). Jensen-Shannon gives 1 bit for both.

These metrics can tell you that two distributions are different, but they cannot tell you *how far apart* they are in any biologically meaningful sense. A small nudge and a complete rearrangement receive the same score.

For single-cell biology, this is not a rare edge case. Biological perturbations shift cell populations continuously — along differentiation trajectories, drug response paths, cell cycle arcs. The question you actually want to answer is not "are these distributions identical?" (they never are) but "how much did the cells move, and where did they go?" That requires a metric that understands what "move" means.

## Wasserstein Distance: The Cost of Rearrangement

Optimal transport starts from a physical metaphor. Think of each cell distribution as a pile of dirt in gene expression space. The Wasserstein distance asks: what is the cheapest way to shovel one pile into the shape of the other?

"Cheapest" means minimizing total work — the amount of dirt moved, multiplied by the distance each bit of dirt has to travel. Moving a cell a short distance costs little. Moving it far costs more. The Wasserstein distance is the minimum total cost across all possible ways of rearranging distribution P into distribution Q.

The key ingredient — and the reason this works where KL and total variation do not — is the *ground metric*: the cost of moving one unit of mass from cell state x to cell state y. This is what encodes the geometry. Nearby cell states are cheap to move between. Distant ones are expensive. The total Wasserstein distance is the minimum total cost — summing mass × distance — across the best possible rearrangement plan.

KL, total variation, and Jensen-Shannon have no ground metric. They compare densities at each location independently, with no notion of how close or far apart those locations are. That missing ingredient is exactly what makes the Wasserstein distance geometry-aware — and exactly what single-cell biology needs.

## Three Scenarios That Show Why This Matters

Return to the treated-versus-untreated comparison. Consider three biologically distinct responses and how each metric handles them.

**Scenario 1: Uniform shift.** The drug gently pushes every cell in the same direction — a subtle, population-wide change in a handful of genes. The means shift slightly. The Wasserstein distance is small, correctly reflecting that each cell moved a little. KL divergence could be large or small depending on arbitrary bandwidth choices in the density estimate — it does not track displacement, so it does not give a reliable answer for small continuous shifts.

**Scenario 2: A rare subpopulation appears.** Most cells are unchanged, but 5% of cells differentiate into a new state far from the original cloud. The mean barely moves — 95% of cells stayed put, so the centroid hardly shifts. The L₂ distance between means misses the event entirely. But the Wasserstein distance sees it clearly: a small fraction of mass traveled a long distance. The cost reflects both how many cells moved and how far they went. The biological event — emergence of a new subpopulation — is visible in the transport cost.

**Scenario 3: Bifurcation.** A single cluster splits into two subpopulations heading in opposite directions. The shifts cancel, so the mean is unchanged. L₂ is zero — completely blind. KL divergence depends on how much the two resulting clusters overlap in density, which depends on dimensionality and kernel bandwidth. The Wasserstein distance captures the full cost: mass moving outward in both directions. It sees the split because it tracks where individual units of probability go, not just what the aggregate density looks like at any one location.

In all three cases, the Wasserstein distance reflects how much biological change actually occurred. Small perturbations give small distances. Large rearrangements give large distances. The metric degrades smoothly instead of blowing up to infinity (like KL when supports do not overlap) or collapsing to zero (like L₂ when the mean does not move).

## Where Wasserstein Is the Wrong Tool: Bulk RNA-seq

Everything above depends on a structural fact that is easy to take for granted: in single-cell data, the scientific objects of interest are the *points*, not the *axes*.

Each cell is a point in gene expression space. The 20,000 gene dimensions define the coordinate system — the space where cells live. But the scientific question is about the points: how did this cloud of cells rearrange? Which subpopulations emerged, split, or disappeared? The gene axes provide the scaffolding. The cells are what you study. Optimal transport works because it compares clouds of points, tracking how mass moves through the space.

Bulk RNA-seq flips this completely. In bulk, the scientific objects of interest are the *axes*, not the points.

A bulk experiment gives you one expression vector per sample — one point in a 20,000-dimensional space, representing the average across millions of lysed cells. A typical differential expression study has three treated and three untreated samples. Those six samples are points in gene space, but nobody is studying the six points as a population. The entire scientific question is: which of the 20,000 axes — which *genes* — changed between conditions? By how much? With what confidence?

The points are replicates. The axes are what you study.

You could formally compute a Wasserstein distance here. Treat each sample as a unit of mass, define a ground metric, and solve the 3 × 3 transport problem. It is trivial to compute. But the resulting number answers a question that no biologist is asking: "how much total work does it take to rearrange three untreated samples into three treated samples?" That cost sums displacement across all 20,000 gene dimensions into a single scalar. Gene identity vanishes in the sum.

This is the key problem. A Wasserstein distance of 50 could mean one gene changed enormously — a sharp, targeted transcriptional response — or it could mean 10,000 genes each shifted by a tiny amount — a diffuse global stress signal. These are completely different biological stories, but the transport cost is identical. The very operation that makes Wasserstein powerful for point clouds — integrating movement across all dimensions — is exactly the operation that destroys the per-gene resolution that bulk experiments demand.

What a biologist needs from a bulk experiment is a ranked list: gene names, log₂ fold-changes, adjusted p-values. Everything downstream — pathway enrichment, GO analysis, network inference, target nomination — requires knowing *which specific genes* drove the difference. DESeq2, edgeR, and limma deliver exactly this. They test each gene individually, model the count distribution, estimate dispersion, and return per-gene effect sizes with statistical confidence. Their entire machinery operates along the axes of the space, one axis at a time.

There is a second problem, just as fundamental. Each bulk RNA-seq sample is not a fixed measurement — it is a *draw from a distribution*. The observed count for any gene in any library is one noisy realization of an underlying biological process: mRNA abundance modulated by transcriptional noise, capture efficiency, sequencing depth, and biological variability between replicates. With n = 3 per group, you have three draws from that distribution for each gene, and the noise is substantial.

This is the essence of what makes DESeq2 and edgeR powerful: they are *model-based statistical inference*. They posit a generative model — the negative binomial — for how counts arise from the underlying true expression level, estimate the parameters of that model (mean expression and dispersion) for each gene, and then ask whether the estimated means differ between conditions more than the estimated noise would predict. Critically, they borrow information across all 20,000 genes to stabilize the dispersion estimates, because no single gene with n = 3 has enough data to estimate its own variance reliably. The model fills in what the small sample size cannot provide.

The Wasserstein distance has no model underneath. It sees three points in one group and three points in another — raw locations in 20,000-dimensional space — and computes a transport cost. There is no generative model, no variance estimation, no borrowing across genes. The distance between two groups of three points is dominated by whatever biological and technical noise happened to land in those six specific libraries, not by the systematic treatment effect. It is a model-free metric applied to a setting where model-based inference is not optional — it is the only way to extract signal from so few observations.

The axes-versus-points distinction makes the boundary crisp. In single-cell data, you have 10⁴ to 10⁵ cells per condition — enough to define a real distribution — and the question is about the points: how did the landscape of cell states rearrange? The transport plan maps cell states to cell states, revealing population-level dynamics that no per-gene test can see. In bulk data, you have three to five samples per condition — far too few to define a distribution — and the question is about the axes: which genes responded? The per-gene test decomposes the difference axis by axis, preserving gene identities that optimal transport erases.

When the scientific question lives along the axes — which genes changed? — use a per-gene test. When the scientific question lives in the point cloud — how did the cells move? — use optimal transport. Applying the wrong tool does not give a wrong answer. It gives an answer to a question nobody asked.

## The Ground Metric: Where the Biology Lives

The most important choice in any optimal transport calculation is one that rarely gets enough attention: the ground metric d(x, y). This is the cost function that defines how expensive it is to move one unit of mass from cell state x to cell state y. Everything else — the transport plan, the total cost, the biological interpretation — flows from this choice.

The default is Euclidean distance in gene expression space. But gene expression data does not live in a flat, uniform space. Cells occupy a low-dimensional curved surface — a manifold — embedded in the high-dimensional space of all possible gene expression vectors. Think of it like the surface of the Earth. New York and London are about 5,500 km apart as the crow flies — a straight line through the planet. But you cannot travel through the planet. The actual distance that matters is the shortest route along the surface: the flight path that follows the curve of the Earth. That surface distance is called a *geodesic* — the shortest path between two points that stays on the surface, rather than cutting through space where you cannot actually go.

The same idea applies to gene expression space. Two cell states that look close in Euclidean distance — a straight line through the 20,000-dimensional space — might sit on different branches of a differentiation tree, separated by a region of expression space that no real cell ever occupies. You cannot travel through that empty region any more than you can fly through the Earth's core. Conversely, two states that seem far apart in straight-line distance might be connected by a smooth developmental trajectory, with intermediate cell states filling in every step along the surface.

This connects directly to Part I of the series. Euclidean distance treats every gene as an independent, equally weighted axis — the same flat-space assumption that Pearson correlation makes. The Mahalanobis correction from Part I reweights by the covariance structure, which helps, but still assumes the geometry is linear and ellipsoidal. On a curved manifold, the right distance is geodesic — the shortest path along the surface, not through the empty space around it.

When the ground metric is geodesic, the Wasserstein distance inherits the manifold structure. Moving mass along a smooth differentiation trajectory is cheap — the path follows the manifold. Moving mass between disconnected branches is expensive — you have to go the long way around through biologically implausible intermediate states, or the cost is infinite if no path exists. The transport plan naturally respects the topology of the biological landscape.

In practice, nobody computes exact geodesics on high-dimensional manifolds. Instead, you build a nearest-neighbor graph — connect each cell to its closest neighbors in expression space — and measure shortest-path distances along the graph. This is what tools like PHATE, diffusion maps, and Waddington-OT do. The shortest path between two cells on the graph follows the chain of intermediate cell states connecting them, approximating the true geodesic along the biological landscape. The result: a ground metric that respects connectivity, branching, and curvature — all the structure that Euclidean distance in the raw 20,000-dimensional space throws away.

## Distance Is Energy

There is something deeper hiding inside the idea of geodesic distance, and it connects the ground metric back to Part I in a way that unifies the entire series.

A geodesic is not just the shortest path — it is the *minimum-energy* path. In physics, moving a particle along a curved surface requires energy, and the geodesic is the trajectory that minimizes total energy expenditure. Distance on a manifold is not really about length. It is about how much energy it costs to get from here to there.

This reframes what the Wasserstein distance actually computes. When you ask "what is the cheapest way to reshape one cell distribution into another?", the "cost" is energy — the total energy required to move all the mass from its current positions to its new positions along the manifold. The Wasserstein distance is the minimum-energy rearrangement.

Now look at how energy shows up mathematically. The Mahalanobis distance from Part I has the form:

d² = (x − μ)ᵀ Σ⁻¹ (x − μ)

This is a quadratic form — xᵀAx — and it is an energy. The vector (x − μ) is the displacement. The matrix Σ⁻¹ defines how expensive it is to move in each direction. The quadratic form computes the total energy of that displacement by taking the dot product of the displacement with its matrix-weighted version: how much does this displacement "push against" the structure of the space?

The same quadratic form appears on the manifold. The graph Laplacian L — the matrix that encodes the neighborhood structure of the cell graph — defines an energy for any function f on the cells:

E(f) = fᵀLf

This measures how much f varies across neighboring cells. A function that is smooth on the manifold (similar values for nearby cells) has low energy. A function that changes abruptly between neighbors has high energy. The Laplacian energy is the manifold equivalent of the Mahalanobis energy — both are quadratic forms, both are dot products weighted by a matrix that encodes the geometry.

The pattern that runs through the entire series is the quadratic form xᵀAx — a displacement, weighted by a matrix that knows the shape of the space, producing a scalar energy. In Part I, A was the inverse covariance Σ⁻¹ and the energy measured how surprising a deviation was in the context of correlated genes. Here, A is the graph Laplacian L and the energy measures how much work it takes to move across the biological landscape. Different matrices, same mathematical structure, same underlying idea: distance is energy, and the matrix tells you what the energy costs.

## The Spectral Connection: Closing the Loop with Part I

This energy perspective leads directly to the eigenspectrum — the thread that has run through every article in this series.

Recall from Part I that the graph Laplacian — the matrix encoding which cells are neighbors of which — has eigenvectors that act like spatial patterns on the manifold. The first eigenvectors capture large-scale structure (the broad separation between cell types). Higher eigenvectors capture increasingly local, fine-grained detail (small density variations within a cluster).

When two distributions are close on the manifold, the Wasserstein distance between them can be decomposed along these eigenvectors. The crucial fact: it weights each eigenvector's contribution by the *inverse* of its eigenvalue. Large-scale patterns (low eigenvalue) get amplified. Fine-grained noise (high eigenvalue) gets suppressed.

This means the Wasserstein distance is naturally tuned to detect the biologically interesting changes — major rearrangements of cell populations across the landscape — while being robust to the uninteresting ones — small local density fluctuations within a cluster. A population that splits into two lineages registers as a large change. Random sampling noise within a stable cluster registers as a small one. The metric's built-in weighting matches biological intuition.

Now compare this to every metric from the series:

- **Euclidean distance** in expression space: weights all directions equally — treats genes as independent, blind to structure
- **Mahalanobis distance** (Part I): weights by inverse covariance eigenvalues — amplifies signal in tightly constrained directions
- **Pearson R** (Part II): weights by covariance eigenvalues — dominated by the high-variance, constitutive directions where baseline expression lives
- **Wasserstein distance**: weights by inverse Laplacian eigenvalues — emphasizes global manifold structure

The pattern is the same across the whole series. Mahalanobis and Wasserstein both amplify signal in geometrically constrained directions by weighting with inverse eigenvalues. The difference is what those eigenvalues describe. Mahalanobis uses the covariance matrix — it captures linear correlations in flat ambient space. Wasserstein uses the Laplacian — it captures the intrinsic geometry of the curved manifold, including connectivity, curvature, and topology that a covariance matrix cannot see.

Wasserstein is the manifold generalization of Mahalanobis. That is the arc of the entire series: from reweighting in flat covariance space (Part I), to variance decomposition against the eigenspectrum (Part II), to transport on curved biological manifolds (here).

## What Optimal Transport Reveals That Other Metrics Miss

To make this concrete, consider a drug response experiment. You treat a cell population with a compound at increasing doses and measure single-cell transcriptomes at each dose. You want to quantify the dose-response relationship.

**Mean shift (L₂ on centroids):** The mean expression vector shifts smoothly with dose. You fit a sigmoid. Clean, interpretable — but you have collapsed 5,000 cells per dose into a single point. If 90% of cells do not respond and 10% respond dramatically, the mean shows a muted shift. The heterogeneity — the fact that the drug works on some cells and not others — is invisible.

**KL divergence:** Sensitive to density changes, but the score depends heavily on how you estimate the density in 2,000 dimensions (nearly intractable), and it cannot distinguish "a few cells moved far" from "many cells moved a little." Both can give the same KL value.

**Wasserstein distance:** At low doses, the transport plan shows most mass staying in place and a small fraction shifting — partial response. At high doses, the entire cloud displaces — uniform response. At intermediate doses, the cloud bifurcates into responders and non-responders, and the transport plan shows exactly where the split happens. The dose-response curve in Wasserstein space captures the full heterogeneity of the response, not just its average.

Better still, the transport plan itself — not just the scalar distance — is biologically interpretable. It maps cell states before treatment to cell states after treatment. This is a cell-level model of what the drug did: which transcriptomic programs activated, which subpopulations were differentially sensitive, and which cells transitioned between identifiable states. No other distance metric produces this kind of output.

## The Series in Full

Looking across all three parts, the series traces a progression from comparing points to comparing distributions, and from flat geometry to manifold geometry.

**Part I.** Compare two expression profiles. Correlation assumes independence and equal weighting across genes. Mahalanobis distance reweights by the covariance eigenspectrum, amplifying signal in the directions where the data is tightly constrained.

**Part II.** Evaluate a predictive model. Whole-transcriptome R is inflated by baseline expression variance. The distinction between observational covariance Σ and causal network propagation N explains why foundation models trained on atlas-scale data achieve high R on held-out observations but do not predict perturbation outcomes. Spectral decomposition separates what models memorize from what they actually learn.

**Part III (here).** Compare two cell populations. Pointwise divergences discard geometry. Wasserstein distance respects the manifold by incorporating the ground metric into the comparison. The spectral weighting — 1/λₖ of the Laplacian — is the manifold analog of the Mahalanobis weighting from Part I.

The unifying principle across the series: naive metrics assume a geometry that biological data does not inhabit. Correlation assumes independence. L₂ assumes flatness. Divergences assume that location does not matter. In each case, the resolution is the same — encode the actual geometry of the data into the metric. Through the covariance eigenspectrum, the Σ-versus-N distinction, or the manifold Laplacian, the mathematical structure is always the same: reweight, decompose, respect the space.

Optimal transport is the most general form of this idea. It takes whatever ground geometry you provide — Euclidean, Mahalanobis, geodesic — and lifts it from a metric on individual points to a metric on entire distributions. The ground geometry determines what the transport cost means biologically. Getting that choice right is where the science lives.

---

*Previous: [Part I — The Distance Your PCA Plot Actually Measures](/posts/beyond-correlation-i-mahalanobis-pca-geometry/) | [Part II — What Foundation Models Actually Learn](/posts/beyond-correlation-ii-foundation-model-eigenspectrum/)*
