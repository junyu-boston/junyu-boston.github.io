---
title: "Variational Inference, Plainly: Why We Approximate the Posterior"
date: 2026-05-16
categories: [Machine Learning, Bayesian Inference]
tags: [variational-inference, reverse-kl, elbo, vae, bayesian-inference, computational-biology]
math: true
---

> This is the post I wish I had read before hearing words like *ELBO*, *reverse KL*, and *variational family*. I am not trying to be complete here. I just want the basic idea to feel concrete.

## The problem variational inference is trying to solve

Suppose you build a probabilistic model with some hidden variable $z$.

- In a mixture model, $z$ might be the hidden cluster label.
- In a single-cell model, $z$ might be a latent cell state.
- In a regulatory model, $z$ might be an unobserved program or factor.

You observe data $x$. What you really want is the **posterior**:

$$p(z \mid x)$$

That posterior answers the question:

> Given the data I saw, what hidden explanations are still plausible?

This is the core Bayesian object. It is not a prediction and not a classifier score. It is the uncertainty that remains after you have seen the data.

The trouble is that in interesting models, the posterior is often too hard to compute exactly.

Not conceptually hard. Computationally hard.

## The simplest intuition

Variational inference is what you do when the posterior you want is too expensive to compute, so you replace it with a simpler distribution that you can actually work with.

You want:

$$p(z \mid x)$$

You use instead:

$$q(z)$$

or sometimes

$$q_φ(z \mid x)$$

where $q$ belongs to some family you chose because it is easy to sample from, easy to evaluate, and easy to optimize.

For one fixed observed dataset, people often write $q(z)$ as shorthand. In amortized settings like VAEs, you will more often see $q_φ(z \mid x)$. Same role, different notation.

Stripped down to the basics, variational inference is just this: choose a family of distributions you can actually work with, then find the member of that family that gets closest to the posterior.

Most of the subject is just the consequences of those two choices.

## A toy example without the jargon

Imagine you are trying to infer which of several hidden biological states could have produced a noisy measurement.

Maybe a cell could be:

- truly quiescent
- weakly activated
- strongly activated

After seeing the data, the true posterior might say:

- quiescent is still somewhat possible
- weak activation is very plausible
- strong activation is also plausible

So the true answer is not a single point. It is a **distribution over explanations**.

Now suppose your approximation family is too simple. Maybe you force yourself to use one Gaussian bump. Then variational inference asks:

> Among all the simple one-bump answers I am allowed to use, which one is the best approximation to the true posterior?

This is why variational inference is approximation from the start. The interesting part is not just the posterior by itself. It is what happens when that posterior has to be represented by a family that may be much simpler than the truth.

## Why not compute the posterior exactly?

Because exact Bayesian inference often requires a normalization constant you cannot evaluate.

The posterior is

$$p(z \mid x) = \frac{p(x, z)}{p(x)}$$

and the annoying term is

$$p(x) = \int p(x, z) \, dz$$

or a giant sum in discrete problems.

That quantity says: add up the contribution of **every possible hidden explanation**. In small models you can do it. In real models you often cannot.

So variational inference avoids direct exact normalization. It says: instead of solving the posterior exactly, solve a nearby optimization problem.

## What gets optimized?

You choose an approximation family $q$ and try to make it close to the true posterior.

The standard variational choice is:

$$D_{KL}(q(z) \parallel p(z \mid x))$$

This is reverse KL with respect to the posterior.

So why this direction?

## Why reverse KL shows up naturally

The expectation is taken under $q$, and $q$ is the object we actually control.

That matters a lot.

If the expectation is under $q$:

- we can sample from $q$
- we can evaluate $\log q(z)$
- we can optimize the parameters of $q$

We usually also know the model well enough to evaluate the joint density up to a constant:

$$\log p(x, z)$$

That is usually enough to make the optimization tractable.

By contrast, forward-KL-style objectives are much less convenient here, because they would require access to expectations under the true posterior itself. Standard VI is practical precisely because it only asks for expectations under $q$, the approximation we chose and control.

So the practical reason for reverse KL is not philosophical. It is mostly about what we can actually compute.

In variational inference, the approximation is the object we can sample from, evaluate, and optimize, so it makes sense that the expectation is taken there.

## Where the ELBO comes from

Most people meet variational inference through the ELBO, which is usually where the explanation starts to feel more complicated than it needs to be.

The ELBO is just another way of writing the same problem.

Start from:

$$D_{KL}(q(z) \parallel p(z \mid x))$$

Expand the posterior:

$$
D_{KL}(q \parallel p) = \mathbb{E}_{q} [\log q(z) - \log p(z \mid x)]
$$

and substitute

$$
\log p(z \mid x) = \log p(x, z) - \log p(x)
$$

Then you get:

$$
D_{KL}(q \parallel p) = \log p(x) - \Big(\mathbb{E}_{q}[\log p(x, z)] - \mathbb{E}_{q}[\log q(z)]\Big)
$$

The big bracket is the ELBO:

$$
\mathrm{ELBO}(q) = \mathbb{E}_{q}[\log p(x, z)] - \mathbb{E}_{q}[\log q(z)]
$$

So maximizing the ELBO is exactly the same as minimizing reverse KL to the posterior, up to the constant $\log p(x)$.

The ELBO is not a separate idea sitting on top of variational inference. It is the same problem written in a way that makes the optimization possible.

## Why VI may look mode-seeking

The part people usually remember is the tendency to lock onto one mode.

Suppose the true posterior has two separated modes, but your variational family only allows one Gaussian. What happens?

Very often, the variational approximation will settle onto one mode rather than spread itself broadly across both.

Why?

Because reverse KL punishes the approximation for putting mass in places where the true posterior is very small. If you smear one Gaussian across the low-density valley between two modes, that can be expensive. So a restricted approximation often prefers to sit cleanly on one plausible region.

That is where the classic “mode-seeking” story comes from.

Two important caveats:

- this is a common behavior, not a universal theorem
- it shows up especially when the approximation family is too restricted to represent the true posterior well

So when people say VI looks mode-seeking, what they usually mean is that a restricted approximation often prefers one credible explanation over a blurry compromise across several explanations.

## A computational-biology example

Suppose you fit a latent-state model to single-cell perturbation data.

For one cell, the posterior over latent states might be genuinely ambiguous:

- state A: recovering
- state B: stressed but viable
- state C: beginning apoptosis

If the posterior is multimodal but your approximation family is a single simple blob, variational inference may lock onto one of those explanations instead of representing the full ambiguity.

That does **not** mean VI is wrong. It means the approximation family was limited.

This matters in biology because missing a posterior mode means under-representing uncertainty about hidden explanations. That is a different problem from generative modeling. Posterior approximation and sample generation are not the same task.

## Named algorithms

If you want a few concrete names to anchor the discussion, these are the ones that come up most often:

- **Mean-field variational Bayes**: assume the latent variables factorize in a simple way, then optimize the factors.
- **Coordinate Ascent Variational Inference (CAVI)**: update one variational factor at a time.
- **Stochastic Variational Inference (SVI)**: a scalable stochastic version used for large models.
- **ADVI**: Automatic Differentiation Variational Inference, which automates much of the optimization.
- **VAE**: the encoder in a variational autoencoder is an amortized variational posterior approximation.

They are all variations on the same move: pick a tractable family $q$, then optimize it as a stand-in for an intractable posterior.

## What variational inference is not

Variational inference is not “the generative version of reverse KL.”

That is too broad and ends up confusing two different problems.

The simpler way to say it is that variational inference is posterior approximation by optimization.

Reverse KL appears naturally because the approximation $q$ is the object we can sample from, evaluate, and optimize.

That is why I would keep this topic separate from the broader question of how generative models are trained. KL shows up in both places, but it is doing different work.

## The takeaway I would keep

If I had to reduce the whole post to one sentence, I would say this: variational inference replaces a hard posterior with an easier distribution and then fits that easier distribution as well as it can.

Everything else, including reverse KL and the ELBO, is just the machinery that makes that approximation practical.

The second thing worth keeping in mind is that when VI looks mode-seeking, that is usually telling you something about the restriction of the variational family, not just about Bayesian inference itself.

For me, that is usually the point where reverse KL stops feeling like jargon and starts feeling like a practical choice.
