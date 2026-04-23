---
title: "How to Read gnomAD Constraint Metrics: pLI, LOEUF, and Missense Z"
date: 2026-04-23
categories: [Computational Biology, Statistics]
tags: [gnomad, genetics, gene-constraint, pli, loeuf, missense-z, graphql, python, statistics]
description: "A practical guide to reading gnomAD constraint metrics: what pLI, LOEUF, and missense Z mean, why they can disagree, and how to query them live from the GraphQL API."
excerpt: "A practical guide to gnomAD constraint metrics for variant interpretation and target evaluation, with a live GraphQL example and a reproducible SCN1A walkthrough."
---

*A gene can be highly intolerant to truncating variants while looking only mildly constrained for amino-acid substitutions. That is not inconsistency. It is what correctly interpreted population genetics looks like.*

A common pattern on a gene report: pLI sits at 1.0 ("extremely loss-of-function intolerant"), while the missense Z-score hovers around 1.3 ("nothing remarkable"). Pasted into a slide, the two numbers look inconsistent. They are not. If you work in variant interpretation, disease genetics, or target evaluation, that distinction matters because these metrics answer different biological questions.

To read these numbers correctly, it helps to step back from the gene page and look at what gnomAD is actually computing. Every gene-constraint metric — pLI, LOEUF, oe_lof, oe_mis, missense Z, synonymous Z — is a version of the same core idea: *observed versus expected under a sequence-aware mutation model*. The metrics differ in which variants they count, how they compute expectation, and how they summarize deviation from expectation. Once that structure is clear, the scores stop looking like disconnected labels and start reading like a coherent decision toolkit.

This post builds up that toolkit from first principles and then shows how to pull the values live for any human gene using the gnomAD GraphQL API.

## If you only remember three things

- **Start with LOEUF for loss-of-function constraint.** Lower LOEUF means stronger depletion of LoF variants relative to neutral expectation.
- **Read missense Z independently.** A gene can be extremely LoF-constrained and only mildly missense-constrained, or the reverse.
- **Always look at the counts behind the score.** `obs` and `exp` tell you how much mutational opportunity sits behind the summary number.

## The mutation rate model

Every constraint metric in gnomAD begins with a per-site mutation rate model. The original formulation is from Samocha et al. 2014 ("A framework for the interpretation of de novo mutation in human disease"). The model treats mutation as a context-dependent process: the probability that a given site mutates from one base to another depends on the local trinucleotide context (one base on each side), with additional adjustments for local factors such as CpG methylation status.

With that per-site, per-alternate-allele mutation rate μᵢⱼ in hand, the expected number of variants of a given functional class in a gene is a sum over sites and possible alternate alleles:

```
E[variants of class C in gene g] = Σᵢ Σⱼ μᵢⱼ · 1{site i → alt j ∈ C}
```

Where C is a functional class — synonymous, missense, or loss-of-function (stop-gained, splice, frameshift). The indicator is just "this specific nucleotide substitution at this position would create a variant in class C given the gene's coding sequence."

This gives you the expected count, *under neutrality*, that you would observe in the gnomAD reference cohort after accounting for sequencing coverage, sample size, and local mutational opportunity. It is the null hypothesis: how many variants would we see if selection did nothing?

The observed count is simply the number of variants of that class actually found in the gnomAD cohort, filtered to the same high-quality calling regions and subjected to the same PASS filters as the expectation.

Once you have expected and observed, the natural comparison is their ratio:

```
oe = observed / expected
```

An oe of 1.0 means the gene behaves exactly like neutral expectation. An oe of 0.2 means the gene is depleted — selection has removed 80% of the variants you would have expected in a cohort this size. An oe above 1.0 means the gene has more variants than expected (rare, usually a sign of a problematic region or a misbehaving model, sometimes real positive selection).

The interesting part is what you do with that ratio once you have it. Three different answers give three different metrics.

## oe_lof, LOEUF, and why the upper bound matters

The raw number oe_lof is simply observed LoF variants divided by expected LoF variants. The trouble is that for short genes, or genes with little mutational opportunity, the expected count can be small — sometimes fractional — and the resulting ratio is noisy. A gene with expected = 2 and observed = 0 has oe_lof = 0, but so does a gene with expected = 20 and observed = 0. The second is much stronger evidence.

LOEUF (Loss-of-function Observed/Expected Upper bound Fraction) solves this by reporting not the point estimate but the upper bound of the 90% confidence interval on the oe_lof ratio (Karczewski et al. 2020, gnomAD v2). The value is computed by profile-likelihood under a Poisson model on the LoF count, then taking the 95th percentile of that profile as the upper end of the 90% two-sided interval.

Why the upper bound? Because the quantity we care about is "how strongly is this gene depleted?" — and we want to be conservative about calling something strongly depleted. The point estimate for a short gene with zero observed LoFs is 0.0, which looks maximally constrained. But if only 2 LoFs were expected, the upper bound of that interval is quite loose, and LOEUF correctly reports something in the 0.4-0.7 range. For a long, mutation-rich gene with zero observed LoFs, the upper bound is tight, and LOEUF correctly reports something like 0.05.

This makes LOEUF a *continuous severity score with built-in statistical confidence*. Lower is more constrained. The gnomAD team recommends using the lowest decile (LOEUF < 0.35 roughly) to flag haploinsufficient candidates, but there is no hard cutoff — it is a continuous ranking.

LOEUF has increasingly replaced pLI as the preferred LoF-intolerance metric. It degrades gracefully on small genes, it is continuous rather than binary-leaning, and it has an obvious statistical interpretation as a conservative upper confidence bound on the oe ratio.

## pLI: the mixture-model ancestor

pLI came earlier (Lek et al. 2016, ExAC) and uses a different statistical construction. It does not report a ratio or a confidence bound directly. Instead, it fits a three-component mixture model over all genes:

- Class 1: genes that are *null-tolerant* — oe_lof is near 1.0. Selection does not filter out LoF variants in this gene.
- Class 2: genes that are *recessive / haploinsufficiency-neutral* — oe_lof is around 0.5. Selection removes some LoF variants, consistent with the gene behaving recessively or being weakly dosage-sensitive.
- Class 3: genes that are *LoF-intolerant* — oe_lof is near 0.1. Selection aggressively removes LoF variants, consistent with strong haploinsufficiency.

For each gene, pLI is the posterior probability that the gene belongs to class 3 given the observed and expected LoF counts. A pLI of 1.0 does not mean "zero tolerance for LoF" in an absolute sense — it means "the posterior probability that this gene falls into the intolerant class is indistinguishable from 1, given the data."

Several consequences follow from this construction:

**pLI ≥ 0.9 is the conventional cutoff for "LoF-intolerant."** This comes from the mixture model: 0.9 is the point at which the posterior is confident enough in class 3 membership to be useful as a screening criterion.

**pLI saturates.** Any gene with enough observed depletion of LoF will end up at pLI = 1.0. The metric cannot distinguish between a gene that is strongly intolerant and a gene that is extremely intolerant. LOEUF can.

**pLI depends on the prior weights of the three classes.** Those priors were fit on ExAC and are now locked in for downstream comparability. This means pLI is a product of its training cohort in a way that LOEUF is not.

**pLI is binary in spirit, continuous in form.** Most practical usage thresholds at 0.9. This is the price of the mixture-model formulation.

A reasonable current practice is: use LOEUF as the primary LoF-intolerance metric, keep pLI around for backwards compatibility with older literature and variant prioritization tools that were tuned against the 0.9 cutoff.

## Missense Z: a different question

The missense Z-score does not fit the mixture-model frame, and it does not report an oe ratio directly. It is a standardized residual.

Under the mutation model, you have an expected count of missense variants for each gene. Across all genes, you also have a relationship between expected and observed missense counts — a mean trend and a variance. The missense Z-score is the number of standard deviations by which a specific gene's observed missense count falls below (or above) the global trend:

```
Z_mis = (expected_mis - observed_mis) / SD_mis
```

Where the sign convention is "positive Z means fewer missense than expected, i.e., missense-depleted." A Z of 3.09 corresponds roughly to p < 0.001 for a one-sided test under approximate normality, and is sometimes used as a conservative cutoff for missense constraint.

A Z of 1.33 means the gene has about 1.3 standard deviations fewer missense variants than the global trend predicts. In a standard normal reference, about 9% of genes have Z ≥ 1.33. It is a mild depletion — noticeable, not extreme.

This is where the apparent contradiction between pLI = 1.0 and Z_mis = 1.33 dissolves:

- **pLI is a posterior probability about LoF class membership.** Saturated at 1.0 means "definitely in the LoF-intolerant class."
- **Missense Z is a standardized residual about missense count.** 1.33 means "mildly missense-depleted."

There is no reason for these to track each other. Some genes are haploinsufficient — lose one LoF-producing copy and the organism is impaired — but tolerate missense variation almost fully, because most missense changes do not destroy protein function. Other genes are the opposite: hypomorphic dominant-negative missense variants are catastrophic, but heterozygous LoF is compensated. The two metrics are not redundant.

A useful mental model: pLI/LOEUF measure *dosage sensitivity at the extreme* — what happens when you delete the protein. Missense Z measures *structural sensitivity on average* — how much residue-level variation the protein tolerates. A gene can be extreme on one axis and middling on the other.

This is why the gnomAD constraint table reports all of them and expects you to read them together.

## The gnomAD v4 cohort

A brief note on the cohort behind these numbers. gnomAD v4 (2024) is built on 807,162 individuals — 730,947 exomes and 76,215 genomes, all aligned to GRCh38. This is roughly 6x the size of gnomAD v2 (125,748 exomes, GRCh37) and is currently the reference cohort for population-level allele frequency and constraint.

For most use cases, query v4 constraint. If you are working with older variant lists still on GRCh37, v2 constraint is still served by the API and remains a legitimate reference — the mutation model and constraint framework are stable across releases, the cohort size is different, and some downstream tools are calibrated against v2 values.

## Pulling constraint live: the gnomAD GraphQL API

The gnomAD browser exposes a public GraphQL API at `https://gnomad.broadinstitute.org/api`. The same endpoint drives the gnomad.broadinstitute.org web interface, which means anything you can see on a gene page can be pulled programmatically with a single HTTP request.

The minimal query for constraint metrics is:

```graphql
{
  gene(gene_symbol: "SCN1A", reference_genome: GRCh38) {
    gene_id
    symbol
    gnomad_constraint {
      pLI
      mis_z
      syn_z
      lof_z
      oe_mis
      oe_mis_lower
      oe_mis_upper
      oe_syn
      oe_syn_lower
      oe_syn_upper
      oe_lof
      oe_lof_lower
      oe_lof_upper
      obs_mis
      exp_mis
      obs_syn
      exp_syn
      obs_lof
      exp_lof
    }
  }
}
```

This returns every constraint number the browser shows, plus their confidence bounds. Note that LOEUF is what `oe_lof_upper` reports — the API exposes the raw field rather than precomputing the "LOEUF" label.

A minimal Python client:

```python
import requests

API = "https://gnomad.broadinstitute.org/api"

QUERY = """
query GeneConstraint($symbol: String!, $ref: ReferenceGenomeId!) {
  gene(gene_symbol: $symbol, reference_genome: $ref) {
    gene_id
    symbol
    gnomad_constraint {
      pLI
      mis_z
      syn_z
      lof_z
      oe_mis
      oe_mis_upper
      oe_syn
      oe_syn_upper
      oe_lof
      oe_lof_upper
      obs_mis
      exp_mis
      obs_syn
      exp_syn
      obs_lof
      exp_lof
    }
  }
}
"""

def fetch_constraint(symbol: str, ref: str = "GRCh38") -> dict:
    r = requests.post(
        API,
        json={"query": QUERY, "variables": {"symbol": symbol, "ref": ref}},
        timeout=30,
    )
    r.raise_for_status()
    payload = r.json()
    if "errors" in payload:
        raise RuntimeError(payload["errors"])
    return payload["data"]["gene"]

if __name__ == "__main__":
    for gene in ["SCN1A", "TP53", "BRCA1", "DMD", "TTN", "CFTR"]:
        data = fetch_constraint(gene)
        c = data["gnomad_constraint"]
        print(
            f"{gene:7s}  pLI={c['pLI']:.3f}  "
            f"LOEUF={c['oe_lof_upper']:.3f}  "
            f"Z_mis={c['mis_z']:.2f}  "
            f"Z_syn={c['syn_z']:.2f}  "
            f"oe_lof={c['oe_lof']:.3f}  "
            f"oe_mis={c['oe_mis']:.3f}"
        )
```

Running this produces a one-line constraint summary for each gene. No API key is required. The endpoint is polite but not unlimited — for batch queries over thousands of genes, space requests out and cache results locally. A reasonable pattern is 5-10 requests per second with retries on 429 and 503.

## Reading a constraint row

Take a single gene and look at the full row. For SCN1A (a well-known haploinsufficient gene in Dravet syndrome), a live GRCh38 query to the gnomAD API on 2026-04-23 returned the following values. Treat them as a reproducible illustration rather than a forever-stable snapshot, since browser releases can update the exact counts.

- pLI = 1.00
- LOEUF (oe_lof_upper) ≈ 0.107
- Z_mis ≈ 8.78
- oe_lof ≈ 0.067
- oe_mis ≈ 0.584
- obs_lof = 13, exp_lof ≈ 193.74
- obs_mis = 1432, exp_mis ≈ 2451.13
- Z_syn ≈ 1.29

What this tells you, as a sequence of nested statements:

**The LoF story.** Expected LoF is about 193.74 in a cohort this size under the neutral mutation model. Only 13 were observed. The point estimate oe_lof ≈ 13 / 193.74 ≈ 0.067 says about 93% of expected LoFs have been removed by selection. The upper bound LOEUF ≈ 0.107 says that even a conservative interpretation still places the gene among strongly LoF-constrained genes. pLI = 1.00 says the mixture model is maximally confident this gene belongs to the haploinsufficient class.

**The missense story.** Expected missense is about 2451.13 in a cohort this size. Observed is 1432. The point estimate oe_mis ≈ 0.584 says about 42% of expected missense variants are missing. Z_mis ≈ 8.78 says this depletion sits far below the global mean across genes and is genuinely extreme. This gene is missense-constrained too, not just LoF-constrained, and both axes agree.

**What the numbers do not say.** They do not tell you *which* missense positions are constrained. A gene can have an overall Z_mis of 8.78 while having exons that tolerate missense freely and domains that tolerate none. Regional constraint scores (gnomAD also publishes these) start answering that. They also do not tell you mechanism — whether depletion is from haploinsufficiency, dominant-negative action, loss of essential binding surface, or developmental lethality. Constraint reports the fingerprint of selection, not its cause.

Now contrast that with a hypothetical pLI = 1.00, Z_mis = 1.33 gene. You would read that as: LoF is strongly selected against (losing a copy matters) but missense is only mildly depleted (residue-level changes are largely tolerated). Mechanistically, this is the signature of a haploinsufficient gene that is not also under strong dominant-negative missense selection. Many transcription factors and dosage-sensitive structural proteins sit here. The pLI and Z numbers are not fighting — they are describing two different slices of the same gene's tolerance profile.

## Batching across a gene list

For a list of candidate genes — say, an associated-gene list from a GWAS or a target-selection shortlist — a simple batching loop handles the volume:

```python
import time
import json
from pathlib import Path

def fetch_many(symbols, out_path="constraint.jsonl", sleep=0.1):
    out = Path(out_path)
    with out.open("w") as f:
        for i, s in enumerate(symbols):
            try:
                rec = fetch_constraint(s)
            except Exception as e:
                rec = {"symbol": s, "error": str(e)}
            f.write(json.dumps(rec) + "\n")
            time.sleep(sleep)
            if (i + 1) % 100 == 0:
                print(f"{i + 1}/{len(symbols)}")
```

Output is JSON-lines, one gene per row, which flows cleanly into pandas for downstream ranking:

```python
import pandas as pd

df = pd.read_json("constraint.jsonl", lines=True)
c = pd.json_normalize(df["gnomad_constraint"])
df = pd.concat([df.drop(columns=["gnomad_constraint"]), c], axis=1)

# Rank by LOEUF (lower = more constrained)
df.sort_values("oe_lof_upper").head(20)

# Or flag candidates with both LoF and missense constraint
candidates = df[(df["oe_lof_upper"] < 0.35) & (df["mis_z"] > 3.09)]
```

The first filter isolates haploinsufficient candidates under the gnomAD-team-recommended LOEUF cutoff. The second adds missense constraint (Z ≥ 3.09, roughly p < 0.001) to identify genes where both classes of variation are under selection — a stronger signal for dosage-sensitive essential genes than either metric alone.

## A few gotchas

**Canonical transcript choice matters.** gnomAD constraint is computed against a canonical transcript. For most genes this is unambiguous, but for genes with multiple biologically meaningful isoforms, constraint for the canonical transcript may not reflect constraint on the isoform you actually care about. The API exposes transcript-level constraint separately — use it when isoforms diverge.

**Coverage biases matter more than they look.** Constraint expectations are adjusted for coverage, but regions of low coverage still have wider intervals. If a gene has poor exome coverage in some exons, its observed counts in those regions are artificially low, and the expected counts are scaled down in response. The net effect on constraint is muted but not zero. Always check that the exons you care about are well-covered before over-interpreting.

**Cohort composition is not uniform.** gnomAD's ancestry composition skews toward European ancestry. This affects rare variant discovery asymmetrically: rare LoFs in ancestries under-represented in the cohort are less likely to be captured. For most genes the effect is small; for some it matters. Pay attention to this when generalizing constraint to a specific clinical population.

**Constraint is a cohort measurement, not a mechanism.** A gene can be constrained because heterozygous LoF causes developmental lethality, because homozygous LoF is lethal and the population never reaches the homozygous state, because the LoF creates a dominant gain-of-function through some downstream pathway, or because it disrupts a gene essential to a highly-selected cell population. Constraint tells you selection acts on LoF — it does not tell you why. Function studies, mouse knockouts, or clinical disease associations are needed to bridge from "constrained" to "mechanism."

## Why these numbers are a toolkit, not a score

The pattern across pLI, LOEUF, and Z is consistent once you see it. Each metric is a statistical statement about observed/expected under a common mutation model. They differ in what they count and how they summarize:

| Metric | Variant class | Summary |
|--------|---------------|---------|
| pLI | Loss-of-function | Posterior probability of haploinsufficient-class membership |
| LOEUF (oe_lof_upper) | Loss-of-function | 90% CI upper bound on oe_lof ratio |
| oe_lof | Loss-of-function | Point estimate of observed / expected |
| Z_mis | Missense | Standardized residual below global missense trend |
| oe_mis | Missense | Point estimate of observed / expected |
| Z_syn | Synonymous | Standardized residual (sanity check — should be near 0) |

Synonymous Z is worth mentioning for calibration. Under a well-calibrated mutation model, synonymous variation should not be under selection, and Z_syn should sit near zero with symmetric distribution. When Z_syn drifts systematically away from zero on a gene, it is usually a sign of technical artifact — low coverage, paralog issues, assembly errors, or a problem with the mutation model for that region. It is a useful sanity check that runs alongside the biological metrics.

The practical workflow for reading constraint on any gene:

1. Start with LOEUF. It is the most information-dense single number for LoF constraint.
2. Cross-check with pLI for legacy comparability — if LOEUF says "top decile" and pLI ≥ 0.9, you are consistent with everything published.
3. Read Z_mis independently, because it is not redundant with the LoF story.
4. Look at obs/exp counts directly when the gene is small. Confidence intervals are meaningful only when you know how much mutational opportunity is behind them.
5. Check Z_syn as a calibration. If Z_syn is far from zero, treat the rest with caution.
6. Remember that constraint describes population-level selection, not individual-variant consequence. A variant in a highly-constrained gene is more likely to be consequential, but constraint is a prior, not a diagnosis.

Read this way, pLI = 1.0 and Z_mis = 1.33 are not contradictory numbers. They are two complementary views of the same gene's tolerance profile: one extreme on one axis, mild on another, entirely consistent with a biology in which some genes are fragile to deletion but robust to point mutation.

The gnomAD constraint framework does not give you a single number that tells you whether a gene matters. It gives you a coordinated set of numbers, each answering a specific question, each with a specific statistical meaning. Reading them together is genetics. Reading any one of them alone is trivia.
