---
title: "KL Divergence Through the Lens of Surprise"
date: 2026-05-06
categories: [Machine Learning, Information Theory]
tags: [kl-divergence, cross-entropy, information-theory, entropy, loss-functions, intuition]
math: true
---

> Adapted from a Chinese blog post by the author of [这篇文章](https://zhuanlan.zhihu.com/p/573385147) on Zhihu. The framing — start from *surprise*, not from *information* — is theirs. I found it the cleanest explanation of KL divergence I have read, so I rewrote it in English for my own notes and left it here in case it is useful to others.

## Why this matters

Most textbooks introduce KL divergence after entropy and cross-entropy, and entropy is introduced as "information content." That order makes the math clean but the intuition opaque. You end up with a correct formula and no feel for what it measures.

This post inverts the order. Start from a single felt quantity — **surprise** — and entropy, cross-entropy, and KL divergence fall out as different expectations over the same thing.

## Step 1: What is surprise?

Let $p(x)$ denote the probability of event $x$. Intuition:

- Low-probability events are surprising when they occur.
- High-probability events are not.

So $1/p(x)$ is a first draft of a surprise measure. But the reciprocal has awkward mathematical properties, so without changing monotonicity we define **surprisal** as:

$$S(x) = \log \frac{1}{p(x)} = -\log p(x)$$

Two bonus properties come for free:

1. A certain event ($p = 1$) has surprisal $0$.
2. For independent events $A$ and $B$, surprisals add: $S(A \cap B) = S(A) + S(B)$.

That's it. Surprisal is the log of the reciprocal of the probability. Nothing deeper.

## Step 2: Entropy is the expected surprise

In most treatments, surprisal is called "information content." That name is Shannon-specific and forces awkward interpretations outside communication theory. "Surprise" transfers better.

If surprisal is a property of a single event, **entropy** is the expected surprisal over the whole distribution:

$$H(X) = \mathbb{E}_{x \sim p}[-\log p(x)] = -\sum_x p(x) \log p(x)$$

or continuously:

$$H(X) = -\int p(x) \log p(x) \, dx$$

Said plainly: **entropy is the average surprise you should expect when sampling from this distribution.**

Two limits are worth knowing:

- **Minimum entropy** — a deterministic event. All mass on one outcome, $H = 0$, no surprise ever.
- **Maximum entropy** — for a continuous distribution with fixed mean and variance, the **normal distribution maximizes entropy**. This is one reason so many natural processes are approximately Gaussian: in a loose sense, nature maximizes surprise subject to constraints. It is also another way to look at the second law of thermodynamics.

## Step 3: Cross-entropy is where frequentist meets Bayesian

The frequentist view: probability is the limit of observed frequency — an objective property of the world.

The Bayesian view: probability is a subjective degree of belief.

These two views are not in conflict. Both can hold simultaneously. Suppose:

- $p_o(x)$ — the **objective** probability of $x$ (o for objective).
- $p_s(x)$ — your **subjective** belief about the probability of $x$ (s for subjective).

Now ask: if reality is $p_o$ but you carry belief $p_s$, what surprise do you expect?

Your surprise at event $x$ is $-\log p_s(x)$ — surprise is measured against *your* model, not reality's. But the frequency with which $x$ actually occurs is $p_o(x)$. So the expected surprise is:

$$H(p_o, p_s) = -\sum_x p_o(x) \log p_s(x)$$

This is **cross-entropy**. Rephrased in words:

> Cross-entropy is the average surprise you experience when you approach an objective random phenomenon carrying a subjective belief about it.

When is cross-entropy large? When $p_s$ assigns low probability to events that $p_o$ says happen often — i.e., when your beliefs are badly mismatched with reality. This is why cross-entropy is the natural loss function in classification: the model's output $p_s$ should match the true label distribution $p_o$, and mismatch costs surprise.

## Step 4: A gambler at a rigged coin table

A numerical example makes the point concrete. Imagine a gambler betting on 100 rounds of a coin flip.

**Scene 1.** The gambler believes heads and tails are each 50%, independent every round, and bets evenly. The dealer is cheating: 99 heads, 1 tail come up. Average surprise per round:

$$-[0.99 \log 0.5 + 0.01 \log 0.5] = \log 2 \approx 0.693 \text{ nats}$$

No subjective preference, so the surprise is the same regardless of outcome.

**Scene 2.** The gambler notices heads is dominating and updates his belief: heads 99%, tails 1%. Next 100 rounds, same rigged outcome (99 heads). Average surprise:

$$-[0.99 \log 0.99 + 0.01 \log 0.01] \approx 0.056 \text{ nats}$$

Beliefs match reality. Surprise is near zero. He feels in control.

**Scene 3.** The dealer flips the rig: now tails comes up 99% of the time. The gambler still believes heads is 99%. Average surprise:

$$-[0.99 \log 0.01 + 0.01 \log 0.99] \approx 4.56 \text{ nats}$$

His worldview collapses. The cost of a wrong model, measured in surprise, is an order of magnitude higher than the baseline.

## Step 5: KL divergence is the excess surprise from being wrong

Cross-entropy measures total expected surprise under a wrong belief. But even perfect beliefs ($p_s = p_o$) produce some surprise, because the event is still random. The floor is $H(p_o)$ itself.

So to isolate the *cost of being wrong*, subtract that floor:

$$D_{KL}(p_o \parallel p_s) = H(p_o, p_s) - H(p_o) = \sum_x p_o(x) \log \frac{p_o(x)}{p_s(x)}$$

This is the **Kullback-Leibler divergence**, a.k.a. **relative entropy**. Key properties:

- $D_{KL} = 0$ if and only if $p_s = p_o$ everywhere.
- $D_{KL} \geq 0$ always (Gibbs' inequality).
- $D_{KL}(p \parallel q) \neq D_{KL}(q \parallel p)$ in general — **KL is not symmetric**, so it is not a true distance metric despite often being called one.

In words:

> KL divergence is the **excess surprise** you pay for holding belief $p_s$ instead of the true distribution $p_o$.

## Step 6: Why ML uses cross-entropy loss instead of KL loss

A common question: if KL is what we actually care about (how wrong is the model?), why do we minimize cross-entropy?

Because when you train a model with parameters $\theta$, the objective distribution $p_o$ is fixed — it comes from the data. So:

$$D_{KL}(p_o \parallel p_\theta) = \underbrace{H(p_o, p_\theta)}_{\text{depends on } \theta} - \underbrace{H(p_o)}_{\text{constant in } \theta}$$

The gradient with respect to $\theta$ is identical whether you minimize KL or cross-entropy. Cross-entropy is computationally identical and notationally simpler. So we use cross-entropy, but the *meaning* is KL.

## Summary

| Quantity | Formula | Plain English |
|---|---|---|
| Surprisal | $-\log p(x)$ | Surprise from a single event |
| Entropy | $H(p) = -\sum p \log p$ | Average surprise under the true distribution |
| Cross-entropy | $H(p_o, p_s) = -\sum p_o \log p_s$ | Average surprise when your beliefs are wrong |
| KL divergence | $D_{KL}(p_o \parallel p_s) = H(p_o, p_s) - H(p_o)$ | *Excess* surprise due to wrong beliefs |

The whole tower stands on one quantity: $-\log p(x)$, the surprise of a single event. Everything else is an expectation taken over a different distribution. Once you see that, the asymmetry of KL is natural — the outer expectation is under reality, not under your beliefs, because reality is what actually generates events.

## Credit

The surprise-first framing above is not mine. It comes from [this Zhihu article](https://zhuanlan.zhihu.com/p/573385147). I translated and reorganized it for my own notes and added the ML-loss connection in Step 6.
