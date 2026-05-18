---
title: "What Changes When You Flip KL Divergence?"
date: 2026-05-15
categories: [Machine Learning, Information Theory]
tags: [kl-divergence, forward-kl, reverse-kl, distillation, sft, generative-models, computational-biology]
math: true
---

> Companion to [KL Divergence Through the Lens of Surprise]({% post_url 2026-05-06-kl-divergence-through-surprise %}). The previous post was about what KL means. This one is about why flipping the order changes the behavior of the model you train.

## Why the asymmetry matters

The important thing about KL divergence is not the formula itself. It is the fact that the order matters.

This is one of those details that looks cosmetic the first time you see it. It is not. In practice, it changes what kind of mistakes the model is pushed to avoid. If you work on biological data, that difference is often the difference between a model that misses a rare but real state and a model that confidently generates something biologically dubious.

In $D_{KL}(P \parallel Q)$, the expectation is taken under $P$. That means the regions where $P$ puts mass are the regions that get inspected. Those are the places where $Q$ is expected to behave well. If you swap the order, you change the standard by which the approximation is judged.

That sounds abstract, but it gets you most of the way there. A lot of the practical behavior people remember about KL, especially mode-covering versus mode-seeking, starts from that one design choice: which distribution gets to decide where the errors count.

## The formula, with the expectation made explicit

$$D_{KL}(P \parallel Q) = \sum_x P(x) \log \frac{P(x)}{Q(x)} = \mathbb{E}_{x \sim P}\left[\log \frac{P(x)}{Q(x)}\right]$$

The expectation is over $P$. So the integrand only matters where $P$ has mass. If $P(x) > 0$ but $Q(x) \approx 0$, the log ratio explodes and the divergence blows up. If $P(x) = 0$, the point contributes zero regardless of $Q(x)$ — so forward KL does not apply a **direct** penalty there.

Said less formally: if $P$ assigns mass somewhere, $Q$ cannot afford to ignore that region. If $P$ assigns no mass there, forward KL does not look there directly. That single fact drives most of what follows.

## Forward KL: $D_{KL}(P_T \parallel P_\theta)$

The expectation is over the **teacher** (or data, or true distribution). The teacher samples points; the student must assign nonzero probability to each one.

- High penalty: teacher has mass, student has none -> log term explodes.
- Low penalty: student over-spreads into regions where the teacher has no mass — these contribute zero.

Result: when the target is multimodal and the model family is limited, the student is often pushed to **cover multiple modes of the teacher**, even at the cost of placing probability mass in low-density valleys between modes. This is the classic **mode-covering** tendency. It is a common variational behavior, not a universal law of every optimization setting.

A computational-biology way to think about it: suppose a single-cell atlas contains five T-cell states: naive, memory, cytotoxic, regulatory, and a rare exhausted population that only appears after treatment. Forward KL says that if the exhausted state is genuinely in the data, the model does not get to quietly drop it because it is rare. A limited-capacity model may respond by spreading probability too broadly, including into the gaps between states. That is the price of insisting on coverage.

In scientific terms, Forward KL is useful when the bigger mistake is failing to represent something real that is present in the data.

## Reverse KL: $D_{KL}(P_\theta \parallel P_T)$

Now the expectation is over the **student**. The student generates samples; the teacher judges whether they are plausible.

- High penalty: student generates something the teacher considers impossible -> log ratio explodes.
- Low penalty: teacher has mass somewhere the student doesn't bother to generate — contributes zero.

Result: when the target is multimodal and the model is constrained, the student is often pushed to **stay inside high-probability regions of the teacher**, even at the cost of ignoring other valid modes. This is the classic **mode-seeking** tendency.

A protein-design example makes the tradeoff vivid. Suppose the teacher distribution represents biologically plausible sequences: foldable, stable, non-toxic, and still consistent with the family you care about. Reverse KL punishes the student for wandering into implausible sequence space: bad hydrophobic patterning, broken active-site geometry, sequences far outside the family, sequences that look fine statistically but would never survive a real screening funnel. The model learns to generate fewer but safer candidates. The cost is obvious too: it may collapse onto one conservative sequence family and miss other valid designs.

Reverse KL is useful when the bigger mistake is inventing structure that should not be there.

## Visual intuition

Here is the cartoon version, for a **constrained unimodal approximator** trying to fit a multimodal target:

```text
True distribution P_T:

       /\\             /\\             /\\
      /  \\           /  \\           /  \\
_____/    \\_________/    \\_________/    \\
```

Under Forward KL, that approximator often spreads out to cover more of the target:

```text
Q under D_KL(P_T || Q): a broader fit

       ______________________________
      /                              \\
_____/                                \\
```

Under Reverse KL, the same approximator often settles on one mode and stays there:

```text
Q under D_KL(Q || P_T): a narrower fit

                      /\\
                     /  \\
____________________/    \\____________________
```

Both are legitimate objectives. They just protect against different kinds of error.

## Why classification cross-entropy is Forward KL in disguise

Suppose the true label for a single-cell profile is T cell, encoded as a one-hot $P = [0, 1, 0]$ over $\{\text{B cell}, \text{T cell}, \text{Monocyte}\}$, and the model predicts $Q = [0.1, 0.7, 0.2]$.

Forward KL collapses to a single nonzero term because $P$ is one-hot:

$$D_{KL}(P \parallel Q) = 1 \cdot \log \frac{1}{0.7} = -\log 0.7$$

That is exactly the cross-entropy loss. So standard supervised classification is minimizing $D_{KL}(P_{\text{label}} \parallel P_{\text{model}})$ — Forward KL with hard targets.

Now try Reverse KL on the same one-hot label:

$$D_{KL}(Q \parallel P) = 0.1 \log \frac{0.1}{0} + 0.7 \log \frac{0.7}{1} + 0.2 \log \frac{0.2}{0}$$

The terms with $P(x) = 0$ in the denominator diverge. One-hot labels are too sharp for Reverse KL — the teacher considers two of three classes literally impossible, but the student gives them nonzero mass, so the penalty is infinite.

This is one reason classification training defaults to Forward KL with hard labels. It is also one reason **distillation uses soft labels**: a teacher distribution like $[0.15, 0.75, 0.10]$ avoids the infinite-penalty pathology and gives you a meaningful full distribution to match. That does **not** make forward and reverse KL interchangeable; it just means both are now mathematically well-defined.

## SFT is exactly Forward KL; on-policy refinement is only Reverse-KL-like

Supervised fine-tuning maximizes log-likelihood on samples drawn from the target:

$$\max_\theta \mathbb{E}_{x \sim P_T}[\log P_\theta(x)]$$

This is equivalent to minimizing $D_{KL}(P_T \parallel P_\theta)$ up to a constant in $\theta$, because

$$D_{KL}(P_T \parallel P_\theta) = \mathbb{E}_{x \sim P_T}[\log P_T(x) - \log P_\theta(x)]$$

and the first term doesn't depend on $\theta$. This part is exact. SFT is maximum likelihood, and maximum likelihood is equivalent to minimizing Forward KL from the data distribution to the model.

Now for the more subtle part. If the student generates its own samples and the teacher can supply a normalized density $P_T(x)$ on the same support, then minimizing Reverse KL gives

$$D_{KL}(P_\theta \parallel P_T) = \mathbb{E}_{x \sim P_\theta}[\log P_\theta(x) - \log P_T(x)]$$

which is equivalent to maximizing

$$\mathbb{E}_{x \sim P_\theta}[\log P_T(x)] + \mathcal{H}(P_\theta)$$

This has the structure of an RL objective: $\log P_T$ acts like a reward from the teacher, and $\mathcal{H}(P_\theta)$ is an entropy bonus discouraging total collapse.

That is the clean mathematical bridge to the usual intuition: reverse-KL-like training pressures a model to stay in regions the teacher scores as plausible. But this is where rigor matters. Real on-policy distillation, preference optimization, and RLHF-style methods usually do **not** optimize literal Reverse KL to a known teacher density. They optimize a reward model, a preference objective, or a policy objective often paired with a KL term to a reference policy. So the right statement is not "RLHF is Reverse KL." The right statement is: **many on-policy objectives inherit a reverse-KL-like flavor because they judge the model on its own samples and punish wandering into bad regions.**

That distinction matters scientifically. SFT and exact Reverse KL are clean textbook objects. Modern post-training pipelines are hybrids. The insight survives, but the identity is approximate, not exact.

If you want concrete names, the cleanest exact examples are from variational inference: **mean-field variational Bayes**, **coordinate ascent variational inference (CAVI)**, **stochastic variational inference (SVI)**, and **ADVI** all use the variational family $q$ to approximate an intractable posterior by minimizing a reverse-KL-type objective. A modern example is the **VAE**, where the inference network is doing variational posterior approximation. On the "reverse-KL-like but not literal Reverse KL" side, the most recognizable example is **PPO-based RLHF**: the policy is evaluated on its own sampled outputs and pushed away from bad regions using reward plus KL regularization, which gives it that same conservative flavor.

That topic is deep enough to deserve its own post. The important point for this page is just to keep the categories straight: reverse KL is most naturally associated with **variational posterior approximation** and with **self-sampled objectives that punish implausible outputs**, not with generative modeling in general.

## Practical summary

| Quantity | Meaning |
| --- | --- |
| $D_{KL}(P \parallel Q)$ | Use $Q$ to approximate $P$ |
| Left distribution | Decides where the expectation is taken |
| Right distribution | Gets penalized for assigning bad probabilities |
| Forward KL: $D_{KL}(P_T \parallel P_\theta)$ | Often mode-covering under constrained approximators |
| Reverse KL: $D_{KL}(P_\theta \parallel P_T)$ | Often mode-seeking under constrained approximators |
| Standard classification | Forward KL with one-hot targets (= cross-entropy) |
| Soft-label distillation | Usually Forward KL / cross-entropy on dense teacher targets |
| SFT / next-token MLE | Exactly Forward KL up to a constant |
| On-policy distillation / RLHF-style | Often Reverse-KL-like, but usually not literal Reverse KL |

## What matters in practice

If the thing you care about is coverage, Forward KL usually matches that instinct better. If the thing you care about is plausibility, Reverse KL usually matches that instinct better.

The intuition is simple once you look at where the expectation is taken. In Reverse KL, the expectation is under the **model itself**. So the loss mainly cares about the places the model actually goes. If the model puts probability mass in some strange region, Reverse KL notices immediately and punishes it. If there is some valid but neglected region that the model never visits, Reverse KL is much less bothered.

That is why Reverse KL feels conservative. It says: before you worry about covering everything that might be real, make sure the things you already generate stay inside territory that looks believable. In biology, that often means preferring a model that produces fewer candidates but keeps them inside known cell-state neighborhoods, known sequence families, or otherwise credible parts of the landscape.

A practical shorthand is this: Forward KL is more sensitive to support that the model fails to cover, while Reverse KL is more sensitive to probability mass the model places in regions the target would consider low-density or implausible.

That is the version I find most useful in computational biology. Sometimes the costly mistake is to miss a rare but real state. Sometimes the costly mistake is to generate something that looks polished mathematically but has no biological business existing. Those are different failure modes, and KL direction lets you decide which one you want the objective to worry about more.

That is also why many real pipelines end up staged rather than pure. First you learn the broad landscape from data. Then you add some mechanism that keeps the model from drifting into regions that are mathematically available but scientifically dubious. In biology, that is often the right order: learn what exists, then become more conservative about what you are willing to generate.
