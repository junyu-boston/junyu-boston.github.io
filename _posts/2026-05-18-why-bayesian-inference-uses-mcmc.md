---
title: "Why Bayesian Inference Uses MCMC"
date: 2026-05-18
categories: [Machine Learning, Bayesian Inference]
tags: [bayesian-inference, mcmc, posterior, uncertainty, variational-inference, computational-biology]
math: true
---

> A short follow-up to the last two posts. If Bayesian inference gives you a posterior over explanations, and variational inference approximates that posterior by optimization, then where does MCMC fit? This is the plain version.

## The goal

In Bayesian inference, the object you want is the posterior:

$$p(z \mid x)$$

This is the distribution of plausible explanations after seeing the data.

If that posterior were simple, life would be easy. You would just write it down, compute whatever summaries you want, and move on.

But in realistic models, the posterior is usually not simple.

It may be:

- high-dimensional
- skewed
- multimodal
- strongly correlated across parameters
- impossible to normalize analytically

So the real bottleneck is not Bayesian philosophy. It is computation.

## Why the posterior is hard to work with

The posterior is

$$p(z \mid x) = \frac{p(x, z)}{p(x)}$$

and the annoying term is the denominator:

$$p(x) = \int p(x, z) \, dz$$

or a giant sum in discrete models.

That denominator says: add up the contribution of every possible hidden explanation.

In small textbook cases, you can do that exactly.
In realistic models, you usually cannot.

So even if you know the model very well, you may still not be able to manipulate the posterior directly.

The crucial trick is that many MCMC methods do **not** need the posterior in fully normalized form. They only need to know it up to proportionality:

$$p(z \mid x) \propto p(x, z)$$

because the annoying constant $p(x)$ does not depend on $z$. This matters because it means you can often evaluate relative posterior plausibility without ever computing the full normalizing constant.

## What MCMC does

MCMC is what you use when you cannot work with the posterior in closed form, but you still want to explore the posterior itself rather than replace it with a simpler approximation.

Instead of saying:

- give me a neat formula for the posterior

MCMC says:

- give me a way to generate samples that spend time in the right regions of the posterior

If you can generate samples whose long-run distribution is the posterior, then you can use those samples to approximate the quantities you care about.

For example:

- posterior means
- posterior variances
- credible intervals
- marginal distributions
- posterior predictive quantities
- probabilities like $P(z_1 > z_2 \mid x)$

So MCMC does not simplify the posterior into one easy family. It tries to approximate the posterior by sampling from it.

## Why the "MC" part?

The Monte Carlo part just means approximation by random sampling.

If I can sample from the posterior many times, then averages over those samples approximate expectations under the posterior.

For example, instead of computing

$$\mathbb{E}[z \mid x]$$

exactly, I can approximate it with the average of many samples:

$$\frac{1}{N} \sum_{i=1}^{N} z^{(i)}$$

You replace an intractable integral with an empirical average.

## Why the "M" part?

The Markov chain part is there because in most problems we cannot directly sample independent draws from the posterior.

So instead we build a stochastic process that moves from one sample to the next in a way that eventually spends time in the posterior with the correct frequency.

The chain wanders through parameter space, but not arbitrarily. It is designed so that, in the long run, the places it visits match the posterior distribution.

So MCMC is not just random guessing. It is a carefully constructed random walk, or in more advanced methods, a more informed guided movement through the posterior landscape.

## A mental picture

Suppose the posterior over two hidden biological states looks like a hilly landscape.

- high hills = highly plausible explanations
- low valleys = implausible explanations

MCMC is like sending a walker into that landscape and recording where the walker spends time.

If the algorithm is working well, the walker spends a lot of time in high-probability regions and little time in low-probability regions.

After a long run, the cloud of visited points tells you the shape of the posterior.

This is part of the appeal. MCMC does not ask you to flatten the landscape into one Gaussian bump if that is not what the landscape looks like. It tries to follow the actual landscape.

## Why people use MCMC

People use MCMC when they want the posterior itself, or at least a faithful sample-based approximation to it.

That usually means one of three things:

1. They care about uncertainty, not just one best estimate.
2. They do not trust a simple parametric approximation.
3. The posterior has structure they do not want to flatten away.

This is especially important when the posterior is asymmetric, correlated, or multimodal.

A point estimate cannot show that structure.
A crude Gaussian approximation may erase it.
MCMC at least tries to preserve it.

## A computational-biology example

Suppose you have a small noisy experiment and want to infer a treatment effect, while also accounting for nuisance parameters and measurement noise.

The posterior may tell you something like this:

- no effect is still possible
- a moderate positive effect is most plausible
- a very large effect is possible but only in a narrow region of parameter space
- the effect size and noise parameter are strongly correlated

One number does not summarize that very well.

If you care about downstream biological decisions, that structure matters.

MCMC lets you inspect that posterior directly through samples. You can look at the marginal distribution of the effect size, the joint distribution with nuisance parameters, or the posterior probability that the effect exceeds a scientifically meaningful threshold.

## MCMC versus variational inference

Variational inference says:

- pick a tractable family $q$
- optimize $q$ to approximate the posterior

MCMC says:

- do not replace the posterior with a fixed simple family if you can avoid it
- sample from the posterior itself by constructing a suitable chain

Variational inference usually buys speed and scalability.
MCMC usually buys faithfulness to the posterior shape.

That is why people often reach for MCMC when uncertainty quantification really matters and the computational budget allows it.

## The classic algorithm names

The main names you will see are:

- **Metropolis-Hastings**
- **Gibbs sampling**
- **Hamiltonian Monte Carlo (HMC)**
- **NUTS** (the No-U-Turn Sampler, a popular adaptive version of HMC)

You do not need to understand all of them at once.

What matters is that they are all different ways to build a chain whose stationary distribution is the posterior.

## Why not always use MCMC?

Because it is expensive.

MCMC can be slow to run, slow to mix, and hard to diagnose in high-dimensional or complicated models. Samples are also correlated, so getting one thousand draws does not mean you got one thousand fully independent pieces of information.

In other words, MCMC is powerful, but it is not free.

That is why variational inference, Laplace approximations, and other faster approximations exist.

## What I would keep in mind

MCMC is used in Bayesian inference because when the posterior is too hard to compute exactly, sampling from it is often easier than solving it in closed form.

I think that is the cleanest way to view it. It is not a different philosophy or a different definition of uncertainty. It is a computational strategy for getting access to the posterior you wanted in the first place.
