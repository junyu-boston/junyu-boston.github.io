---
title: "DESeq2 lfcThreshold: Why Filtering by Fold Change After the Test Is Not Equivalent"
date: 2026-04-24
categories: [Computational Biology, Statistics]
tags: [deseq2, rna-seq, differential-expression, hypothesis-testing, wald-test, fdr, equivalence-testing]
description: "A plain-English tour of DESeq2's lfcThreshold and altHypothesis arguments, why post-hoc fold-change filtering changes the claim without changing the test, and what equivalence testing does that naive filtering cannot."
excerpt: "Filtering by fold change after a default Wald test feels equivalent to lfcThreshold. It isn't — and the difference shows up as thresholded claims that the original padj does not support, plus the inability to claim a gene is confidently unchanged."
---

*A gene estimated at log2FC = 0.6 with standard error 0.3 passes a "fold change > 0.5" filter. The same estimate, fed into `lfcThreshold=0.5`, does not. Both are correct — they answer different questions.*

If you have ever run DESeq2, you have probably written something like this:

```r
res <- results(dds)
hits <- subset(res, padj < 0.05 & abs(log2FoldChange) > 0.5)
```

It looks clean. Intuitive. A natural blend of statistical rigor (adjusted p-value) and biological meaning (fold-change threshold). I've written it a hundred times. But it does something subtle: it answers two *different* questions in one pipeline.

DESeq2 has a built-in tool for the right way to do this: the `lfcThreshold` and `altHypothesis` arguments to `results()`. They move the threshold into the test itself, so you are asking one coherent question from start to finish, rather than asking one question and then filtering on a different criterion.

This post walks through what that distinction actually costs you, why reviewers care about it, and how to use `lfcThreshold` correctly for four common questions in genomics.

This post walks through what `lfcThreshold` actually does, the four flavors of `altHypothesis`, and the specific ways that filtering after a default test fails.

## If you only remember three things

- **The threshold belongs in the test, not in the filter.** `lfcThreshold=x` tests whether β is significantly outside the ±x band. Post-hoc filtering tests whether β is significantly different from zero, then separately asks whether the point estimate happened to clear x.
- **The reported FDR guarantee does not transfer to the new thresholded claim.** `padj` controls FDR for the null you tested. If you change the claim afterward by filtering on effect size, that 5% guarantee no longer applies to the set you report for `|β| > x`.
- **Within DESeq2, `lessAbs` is the built-in way to claim a gene is unchanged within a margin.** No amount of filter gymnastics on a default test gives you equivalence testing. The null has to encode "change is large"; a small p-value then rejects that in favor of "change is small."

## The default test answers a specific, narrow question

When you call `results(dds)` without arguments, DESeq2 runs a Wald test for each gene with the null hypothesis

> H₀: β = 0

where β is the log2 fold change. The test statistic is β̂ / SE(β̂). A small p-value means the estimate is far enough from zero relative to its uncertainty that you can reject "no change."

This is a very sharp question. With enough samples, the standard error shrinks, and β̂ = 0.05 becomes statistically reliable — a change far too small to matter biologically. The test is doing its job; the job just is not the one a biologist asked for.

Biologists usually want to ask one of these instead:

- Which genes changed by at least 1.5-fold, up or down?
- Which genes went up by at least 50%?
- Which genes can I confidently say did *not* change?

The first two shift the effect-size bar away from zero. The third flips the question entirely — it asks for evidence of the null, not evidence against it. Neither is a pure sample-size question, and neither is what the default test answers.

## `lfcThreshold` moves the goalposts honestly

With `lfcThreshold = x`, DESeq2 reformulates the null. Depending on `altHypothesis`, the null becomes:

| `altHypothesis` | Alternative (what you want to find) | Null (what the p-value rejects) |
| --- | --- | --- |
| `greaterAbs` | \|β\| > x | \|β\| ≤ x |
| `lessAbs` | \|β\| < x | \|β\| ≥ x |
| `greater` | β > x | β ≤ x |
| `less` | β < -x | β ≥ -x |

The key phrase from the DESeq2 vignette: *the alternative hypothesis is specified by the user — the genes you are interested in finding — and the test provides p-values for the null, the complement.*

Read that twice. `altHypothesis="lessAbs"` means you are interested in flat genes. The null is "this gene changed a lot." A small p-value rejects the null in favor of "this gene is confidently flat." The test machinery is unchanged; only the null region moves.

## The four flavors, with plain-English questions

**`greaterAbs` — \|β\| > x.** The workhorse. "Which genes changed by more than my threshold, either direction?" Blue dots on the MA-plot sit outside the ±x band.

**`lessAbs` — \|β\| < x.** Equivalence testing. "Which genes are confidently unchanged?" The implementation computes two one-sided tests (β significantly below +x *and* β significantly above -x) and takes the max p-value. Both must be small for the gene to pass. Blue dots sit inside the band — and they concentrate at high counts, because you need precision to prove something is flat.

**`greater` — β > x.** One-sided up. "Which genes were induced by more than x?" Repression is ignored entirely; a gene that collapsed tenfold is not flagged.

**`less` — β < -x.** One-sided down. Mirror image. Useful for siRNA or ASO knockdown experiments: "Which transcripts were knocked down by at least 50%?"

## Why post-hoc filtering is not equivalent

Here is the failure mode. Consider two genes, both estimated at β̂ = 0.6.

- **Gene A:** β̂ = 0.6, SE = 0.1. The 95% interval is roughly 0.4 to 0.8. Confidently above 0.5.
- **Gene B:** β̂ = 0.6, SE = 0.3. The 95% interval is roughly 0.0 to 1.2. Might really be 0.1.

Both genes have a default Wald test p-value against H₀: β = 0. Gene A's p-value is tiny (0.6 is six standard errors from zero). Gene B's p-value is also small, maybe 0.05 (0.6 is two standard errors from zero). Both pass `padj < 0.05`. Both have point estimates above 0.5, so both pass your `abs(log2FoldChange) > 0.5` filter.

Now run the same data with `lfcThreshold=0.5, altHypothesis="greaterAbs"`. The test statistic becomes roughly (β̂ - 0.5) / SE, reflecting how far the estimate is from the threshold, not from zero.

- Gene A: (0.6 - 0.5) / 0.1 = 1.0. Borderline — the estimate is only one SE above the threshold. P-value around 0.15.
- Gene B: (0.6 - 0.5) / 0.3 = 0.33. Not close to significant. P-value around 0.37.

Notice what happened. Gene A *dropped* under the honest test, because while its estimate is tightly determined, it is only marginally above 0.5. Gene B drops even harder. The post-hoc filter was declaring victory for both; `lfcThreshold` correctly says neither gene provides strong evidence of effect beyond the threshold.

This is the general pattern. **Post-hoc filtering keeps genes whose point estimates happen to clear the threshold, regardless of whether the data actually support clearing it.** The uncertainty is ignored at the filtering step because the test that computed the uncertainty was asking a different question.

## The FDR gotcha: it does not travel with you

This is the part that matters most in review, and it is easy to miss in practice.

DESeq2's Benjamini-Hochberg adjustment — the `padj` column — controls the false discovery rate **for the null hypothesis that was tested**. When the null is β = 0, `padj < 0.05` means: among the genes I am calling significant for β ≠ 0, no more than 5% are expected to be false positives for β ≠ 0.

Now you filter on `|log2FoldChange| > 0.5`. You are reporting a different set of genes, for a different claim (evidence of |β| > 0.5). The original BH-adjusted `padj` no longer directly justifies that thresholded claim.

Concretely, some genes in your filtered set will be:

- Truly nonzero (so they were real β ≠ 0 discoveries) but with |β| < 0.5 in truth, whose point estimates happened to land above 0.5 by sampling noise.

These are false positives for the claim you are reporting. They were not false positives for the test you ran. The filter picked them up silently, and the original 5% FDR statement applies to the β ≠ 0 claim, not automatically to the thresholded claim you are now presenting.

With `lfcThreshold=0.5`, the `padj` column is computed against the correct null, and the FDR claim is honest for the set you report.

This matters more than it sounds. One recurring pattern in DE analyses is to filter after the default test and then cite the adjusted p-value next to an effect-size claim. In that situation, the reported FDR number is tied to the original null, not to the thresholded claim now being presented.

## Equivalence testing: the filter-after method cannot do this at all

If you want to identify genes that are *confidently unchanged* — say, to validate that housekeeping genes are stable, or that off-target genes were not perturbed by your compound — there is no post-hoc filtering workaround.

Consider two genes:

- **Gene C:** β̂ = 0.1, SE = 0.05. Tight, truly close to zero.
- **Gene D:** β̂ = 0.1, SE = 2.0. Wildly uncertain; could be anything.

Both have |β̂| < 0.5. A naive filter would keep both as "unchanged." But Gene D provides no evidence of anything; its interval spans enormous effects in either direction.

The default test offers no help here. A large p-value for H₀: β = 0 is not evidence that β = 0 — it is absence of evidence, which is famously not the same thing. Gene D has a large p-value precisely because its uncertainty is so large. You cannot distinguish Gene C from Gene D by looking at p-values against zero.

`altHypothesis="lessAbs"` solves this cleanly. Its null is "|β| ≥ 0.5." Gene C rejects that null (the tight interval excludes ±0.5). Gene D does not (the sprawling interval easily contains values outside ±0.5). Only Gene C passes.

This is classical equivalence testing — the two one-sided tests, TOST, procedure — dropped into the DESeq2 framework. Within DESeq2's built-in `results()` workflow, it is the correct way to make a claim about genes that are unchanged within a specified margin.

## Where filter-after still works

There are real situations where the two-step filter is perfectly fine:

- **Exploratory browsing.** Mining results in a Shiny app, looking for candidates to chase, ranking by effect size — point-estimate filtering is fast and sensible. You are not writing a methods section.
- **Well-powered studies at scale.** If you have thousands of samples, every gene's standard error shrinks, and "β̂ > 0.5" and "evidence for β > 0.5" become almost the same thing. The two methods converge.
- **When FDR is not your claim.** If you write "these are interesting genes for follow-up" and never invoke the FDR number, the lack of correspondence does not matter. The statistics is lighter duty.

But the moment your published methods or results section says something like *"genes with log2 fold change > 0.5 at FDR < 0.05"* — use `lfcThreshold`. It costs nothing, it is the honest statement, and it takes one line to change.

## A minimal worked example

```r
library(DESeq2)

# Standard setup elided — assume dds is ready
# Default test: which genes are reliably nonzero?
res_default <- results(dds)

# The common but wrong pattern
hits_wrong <- subset(res_default,
                     padj < 0.05 & abs(log2FoldChange) > 0.5)

# The correct pattern for "meaningfully changed"
res_correct <- results(dds,
                       lfcThreshold = 0.5,
                       altHypothesis = "greaterAbs")
hits_correct <- subset(res_correct, padj < 0.05)

# Equivalence: which genes are confidently unchanged?
res_flat <- results(dds,
                    lfcThreshold = 0.5,
                    altHypothesis = "lessAbs")
flat_genes <- subset(res_flat, padj < 0.05)

# Directional: which genes were knocked down?
res_down <- results(dds,
                    lfcThreshold = 0.5,
                    altHypothesis = "less")
knockdowns <- subset(res_down, padj < 0.05)
```

All four of these `results()` calls are cheap — they reuse the estimated coefficients and standard errors from the original fit, changing only the hypothesis and multiplicity adjustment. The real work is upfront: deciding what question you are asking before you look at the data.

## The takeaway

If the threshold matters to your biological claim, the threshold has to be in the test, not in the filter.

Here's the practical version: decide what question you are asking *before* you run `results()`. Pick your `altHypothesis` and `lfcThreshold`. Then report the `padj` from that test.

If you layer a fold-change filter on a default p-value and cite them together, the FDR number is bonded to the wrong null hypothesis. Anyone reviewing the analysis carefully will catch it. And if they do not, you will know it is there.

## References

- Love, M. I., Huber, W., and Anders, S. (2014). Moderated estimation of fold change and dispersion for RNA-seq data with DESeq2. *Genome Biology* 15:550.
- The DESeq2 vignette section "Tests of log2 fold change above or below a threshold" — where the four `altHypothesis` options and the MA-plot panels live.
- Schuirmann, D. J. (1987). A comparison of the two one-sided tests procedure and the power approach for assessing the equivalence of average bioavailability. *Journal of Pharmacokinetics and Biopharmaceutics* 15:657-680. — origin of the TOST framework that `lessAbs` implements for count data.
