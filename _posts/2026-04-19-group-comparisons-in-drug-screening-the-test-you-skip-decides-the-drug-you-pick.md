---
title: "Group Comparisons in Drug Screening: The Test You Skip Decides the Drug You Pick"
date: 2026-04-19
categories: [Statistics]
tags: [anova, multiple-comparisons, effect-size, drug-screening, computational-biology, post-hoc, kruskal-wallis]
math: true
---

A screening campaign can test hundreds of compounds, but the decision point keeps repeating: control versus treated, low dose versus high dose, one cell line versus another. Each time, the analysis asks a simple question: "are these groups different?" The test used to answer that question helps shape which compounds look promising and which ones quietly drop out.

The usual failure is not that scientists forget statistics. It is that they ask one scientific question and run the test for a different one.

## Start with the actual decision

Consider a typical primary follow-up experiment in oligonucleotide screening. A candidate is tested in four conditions: untreated control, low dose, medium dose, and high dose. Each condition has 3-5 biological replicates.

Most people immediately reach for pairwise t-tests:

- control vs low
- control vs medium
- control vs high

If any comparison is significant, the compound is labeled dose-responsive.

That shortcut is attractive, but it bundles together three different scientific questions:

1. Do any of the four groups differ at all?
2. Is each dose different from control?
3. Is there a monotone dose trend?

These are not the same question, and they should not default to the same test.

## Why uncorrected pairwise testing causes trouble

If you run three comparisons at $\alpha = 0.05$ and treat each result independently, the chance of at least one false positive rises above 5%.

Under independence, the family-wise error rate is:

$$
\operatorname{FWER} = 1 - (1 - \alpha)^m = 1 - (0.95)^3 = 0.143
$$

That 14.3% is a back-of-the-envelope calculation under independence. In shared-control designs the exact value changes because the comparisons are correlated, but the practical message survives the nuance: running several uncorrected pairwise tests makes false positives more likely.

If this pattern is repeated across many compounds, some compounds will advance because of multiplicity alone rather than biology.

That is the real problem. The screen starts ranking statistical accidents.

## The practical decision tree

For screening scientists, the most useful rule is not "ANOVA versus t-test." It is:

**Match the test to the decision you need to make.**

```text
2 independent groups?
├── Default: Welch's t-test
└── If data are very skewed, ordinal, or clearly outlier-driven with tiny n: Wilcoxon rank-sum

3+ independent groups?
├── Asking "do any groups differ?"
│   ├── Default: Welch ANOVA
│   └── Follow with multiplicity-adjusted contrasts if needed
├── Asking "which doses differ from control?"
│   └── Use many-to-one contrasts (for example, Dunnett-type adjustment)
├── Asking "is there a dose trend?"
│   └── Use a trend test or regression on dose
└── If data are strongly non-Gaussian or rank-like with very small n
    ├── Kruskal-Wallis for the global question
    └── Dunn's test for adjusted pairwise follow-up
```

This tree looks slightly more complicated than "just run ANOVA," but it maps much better to how screening decisions are actually made.

These rank-based methods are fallbacks, not universal replacements for unequal-variance or bounded-outcome problems. If the assay readout has a clear data-generating structure, use a model that respects it.

## The most common screening questions

### 1. "Do any of these groups differ?"

This is the omnibus question. You do not yet care which pair is different; you want to know whether the experiment shows evidence of any group effect.

For that question, ANOVA is appropriate. In practice, **Welch ANOVA** is often the safer default than classical ANOVA because assay variance is frequently unequal across groups.

If your first question is global and the global test is significant, you can then ask which contrasts matter.

### 2. "Is each dose different from control?"

This is the question many screens actually care about. You are not equally interested in low vs medium or medium vs high. You want treated-versus-control comparisons.

That is a **many-to-one** problem, not an all-pairs problem. A Dunnett-style adjustment is designed for exactly this setting and is usually more efficient than testing every possible pair.

If you instead run all-pairs tests, you pay a power penalty for comparisons you never needed.

### 3. "Does response increase with dose?"

This is not the same as asking whether any pair of groups differs. It is a trend question.

If the scientific claim is dose dependence, use a method that encodes dose order: a trend test or a regression model with dose as an ordered predictor. That approach is often more aligned with the biology than a pile of separate pairwise tests.

This matters because a real gradual trend can be missed by pairwise testing when no single jump is large enough on its own.

## Do not let normality be the only decision rule

Many statistics guides teach a simplified branch:

- normal data -> t-test / ANOVA
- non-normal data -> Wilcoxon / Kruskal-Wallis

That is useful as a first approximation, but it is incomplete for screening work.

With 3-5 replicates per group, formal normality tests have low power. The bigger practical issues are often:

- unequal variance across groups
- outliers
- bounded readouts such as percent viability or percent knockdown
- plate or batch effects
- non-independent replicates

So the real workflow is not "run a normality test and let software choose." It is:

1. Inspect the data.
2. Ask what scientific contrast matters.
3. Check whether replicates are genuinely independent.
4. Use a method whose assumptions are at least approximately compatible with the assay.

For bounded or highly asymmetric outcomes, a data-type-specific model may be better than forcing everything into a Gaussian framework. The main point is conceptual: do not let convenience decide the model.

## Before worrying about normality, rule out pseudoreplication

Before worrying about p-values, check what a replicate actually is.

If three wells came from the same transfection mix on the same plate, they are not three independent biological replicates in the strongest sense. If the same compound was measured across several plates or days, plate and batch can explain part of the variation.

This matters because inflated sample size creates artificially small standard errors. Once that happens, even the "correct" test will produce misleadingly small p-values.

In screening, the fastest route to a false hit is often not the wrong test. It is treating technical structure as biological replication.

## Post-hoc testing: use it, but use it for the right family

Multiplicity correction is not optional once you ask several questions of the same experiment.

The key point is to define the family of comparisons first.

- If you want **all pairs**, Tukey-type adjustment makes sense.
- If you want **each dose vs control**, Dunnett-type adjustment is a better match.
- If you planned a small set of specific contrasts in advance, those contrasts can be tested directly with appropriate adjustment.

The mistake is not "running post-hoc tests without ANOVA" in the abstract. The mistake is asking many questions and pretending you only asked one.

## Effect size is what makes a hit worth chasing

A p-value answers: "is this signal hard to explain by random variation under the null?" It does **not** answer: "is this signal large enough to matter biologically?"

That second question is what decides whether a screening hit is worth follow-up.

For bench teams, the most interpretable effect size is usually the **raw assay-scale effect**:

- 8 percentage points more knockdown
- 15 percentage points lower viability
- 0.3 log units higher reporter signal

That quantity is easier to reason about than a standardized effect size alone.

### What to report in practice

For a screening comparison, a good minimal package is:

1. The estimated mean or median difference
2. A confidence interval
3. The adjusted p-value
4. A standardized effect size if cross-assay comparison is useful

If you need a standardized effect size for small samples, **Hedges' g** is usually preferable to raw Cohen's d because it corrects the small-sample bias.

If you used a rank-based analysis, use a rank-based effect size rather than forcing everything into Cohen's d.

### Pre-specify what counts as meaningful

Every assay has a practical effect floor below which a difference is statistically interesting but operationally irrelevant.

That threshold should be decided before you look at the result. For example:

- "Below 10 percentage points of knockdown improvement, we do not reprioritize compounds."
- "Below 20% viability separation, the assay is too noisy for ranking."

The exact number depends on assay reproducibility and downstream cost, but the principle is universal: **define meaningful effect size before inference, not after.**

As discussed previously in the [Decomposing Significance](/posts/decomposing-significance-iii-effect-size-vs-variance/) series, significance can be driven by variance as much as by effect magnitude. The screening translation is straightforward: a compound can look impressive because the assay happened to be quiet, not because the biological effect was large.

## Two examples that screening scientists will recognize

### Example 1: The compound that looks better than it is

Suppose control, low, medium, and high dose differ by only small absolute amounts, but the wells are unusually tight in one run. Uncorrected pairwise testing may produce one or two nominally significant comparisons.

After multiplicity adjustment, the evidence weakens. Then you look at the raw effect size and realize the best surviving contrast is only a few percentage points.

That is not a strong hit. It is a precise measurement of a small effect.

### Example 2: The gradual dose response

Now consider a compound with a steady improvement from control to low to medium to high. No single neighboring step is dramatic, so pairwise testing can look underwhelming.

But a global group test or a dose-trend model may detect a consistent ordered pattern.

That compound is often more interesting biologically than the one with one isolated nominally significant comparison.

The lesson is simple: the wrong test can hide a real trend or exaggerate a trivial one.

## High-throughput screens have two multiple-testing layers

Once you move from one compound to a campaign, there are usually at least two multiplicity problems:

1. **Within a compound**: multiple doses, time points, or pairwise contrasts
2. **Across the screen**: many compounds, often across several endpoints or models

Correcting only within a compound does not solve the whole problem.

In some pipelines, the primary screen is used only for ranking and a confirmatory assay provides stricter control later. In others, the primary screen itself is expected to control false discovery rate across the compound set.

Either strategy can be reasonable. What matters is that the analysis plan states which error rate is being controlled at which stage.

## A minimal screening checklist

Before calling a group difference real, ask:

1. What is the actual scientific question: any difference, dose vs control, or dose trend?
2. Are the replicates independent, or am I counting technical structure as biology?
3. Are variances similar enough, or should I use a Welch-type method?
4. Am I correcting for the full family of comparisons I actually asked?
5. Is the effect large enough to matter in this assay?
6. If this is one compound among many, where is across-screen multiplicity handled?

If a screening analysis cannot answer those six questions clearly, the p-value is not the problem. The experimental decision rule is.

## Final takeaway

In screening, the first statistical mistake is usually not an exotic one. It is asking a dose-trend question with an all-pairs test, a dose-vs-control question with an omnibus test alone, or a biological-priority question with only a p-value.

Match the test to the decision. Match the effect size to the assay. Match the multiplicity correction to the family of comparisons you actually made.

That is not statistical polish. It is part of hit triage. The test you choose to run — or skip — still helps decide the drug you pick.
