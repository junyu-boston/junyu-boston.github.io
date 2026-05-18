---
title: "Bayesian Inference Is a Distribution Over Explanations"
date: 2026-05-17
categories: [Machine Learning, Bayesian Inference]
tags: [bayesian-inference, posterior, uncertainty, probabilistic-modeling, classical-ml, computational-biology]
math: true
---

> A short note on the idea that made Bayesian inference finally click for me: the answer is usually not a point. It is a distribution over plausible explanations.

## Why this idea matters

A lot of machine learning is taught as if the goal were always to predict the one right answer.

- What class is this image?
- What value should this model output?
- Which candidate should I rank first?

That framing is useful, but it hides something important.

In many real problems, especially in biology, the data do not uniquely determine one clean explanation. They narrow the field. They rule some explanations out, make others more plausible, and leave a few still in contention.

That is the mindset shift in Bayesian inference.

The goal is not just to say what the best explanation is. The goal is to represent **how plausible each explanation remains after seeing the data**.

## The core Bayesian object

Suppose you observe data $x$ and there is some hidden thing you care about, call it $z$.

- a hidden cell state
- a cluster assignment
- a latent regulatory program
- a kinetic parameter
- a treatment effect size

In Bayesian inference, the main object is the posterior:

$$p(z \mid x)$$

This is the distribution of plausible explanations **after** seeing the data.

That is why I find the posterior more intuitive than many textbook introductions make it sound. It is not mystical. It is just updated uncertainty.

Before the data, you have uncertainty about $z$.
After the data, you still have uncertainty about $z$, but it is no longer the same uncertainty.

Some explanations become much more plausible. Some collapse. Some remain ambiguous.

That updated uncertainty is the posterior.

## A simple example

Imagine a noisy single-cell measurement. You observe one cell with a transcriptomic profile that could reasonably fit more than one biological story.

Maybe the cell is:

- quiescent but slightly stressed
- early activated
- transitioning toward exhaustion

The data may strongly suggest that one story is more likely than the others. But unless the measurement is perfect and the biology is perfectly separated, it usually does not force absolute certainty.

So the Bayesian answer is not:

> the cell is early activated

The Bayesian answer is closer to:

> given the data, early activation is the most plausible explanation, stress is still possible, and exhaustion is less likely but not ruled out

That is a distribution over explanations.

This is the point that matters. The posterior is not just giving you a winner. It is telling you how much ambiguity remains.

## What changes compared with classical machine learning?

The cleanest contrast is not "Bayesian versus all of machine learning." That would be too sloppy.

The cleaner contrast is this:

- standard classical workflows often aim for a **point estimate**
- Bayesian inference treats **uncertainty itself as part of the answer**

In classical regression, you often fit one best parameter vector and then make one prediction.
In classical classification, you often ask for the most likely label.
In classical optimization, you often want the single best setting.

That workflow is usually:

1. pick the best parameters
2. use them as if they were the truth
3. output a point prediction

Bayesian inference takes a different view.

Instead of pretending the unknown quantity has already been pinned down exactly, it keeps track of residual uncertainty. The answer is not just where the model lands. It is how widely the model should still hedge.

## Point estimate versus posterior

A point estimate says:

> this is the one value I will use

A posterior says:

> these values are still plausible, but not equally plausible

That sounds like a small difference. It is not.

Once you keep a full posterior instead of one point estimate, several things change immediately:

- you can tell the difference between confidence and ambiguity
- you can separate strong signal from weakly supported guesses
- you can propagate uncertainty downstream instead of hiding it
- you can ask not just "what is best?" but also "how sure am I?"

In biology, that last question matters a lot.

If two mechanistic stories fit the data almost equally well, pretending there is only one answer is not a strength. It is information loss.

## Why this matters more in biology than in clean benchmark datasets

In many benchmark machine-learning problems, the hidden assumption is that there really is one crisp target and that enough data will eventually reveal it.

Biology is often messier.

Measurements are noisy. Cell states blur into one another. Mechanisms overlap. Experimental readouts are indirect. Replicates are limited. Sometimes several explanations are genuinely consistent with the data you have.

That is exactly where the Bayesian mindset helps.

It gives you permission to say:

> the data are suggestive, but they do not uniquely identify one explanation

That is not a failure of inference. That is often the honest scientific answer.

## Prediction is still possible

One reason Bayesian language can sound abstract is that people hear "distribution over explanations" and think it replaces prediction.

It does not.

Bayesian inference still lets you predict. It just changes what prediction means.

Instead of making a prediction from one fixed best parameter value, you average over the explanations that remain plausible.

That is the idea behind the posterior predictive distribution.

In plain language:

> do not predict as if one explanation were certainly true when the data still support several

This is a much more natural way to think when the system is noisy or partially observed.

## A concrete contrast

Suppose you are estimating a drug response slope from a small noisy experiment.

A classical workflow might say:

- estimate one slope
- report that slope
- move on

A Bayesian workflow says:

- low slopes are possible
- moderate slopes are most plausible
- very steep slopes are less plausible but maybe still supported

Now imagine you must decide whether to spend money on a follow-up experiment.

Those two outputs are not the same.

A single point estimate hides whether the evidence is sharp or fragile.
A posterior makes that visible.

That is why Bayesian inference is not only about being mathematically elegant. It is about preserving the structure of uncertainty instead of flattening it too early.

## One important caveat

This does **not** mean classical machine learning is wrong, or that classical methods never represent uncertainty.

They often do. Probabilistic classifiers, conformal methods, quantile regression, ensembles, and calibrated predictive intervals all try to recover part of that picture.

So the difference is not that classical ML predicts points and Bayesian methods predict distributions, full stop.

The more precise statement is this:

> standard non-Bayesian workflows often center a single fitted solution, while Bayesian inference makes uncertainty over hidden quantities a first-class object.

That is the distinction worth keeping.

## The sentence I would keep

If I had to compress the idea into one sentence, it would be this:

> Bayesian inference does not ask only "what is the best explanation?" It asks "after seeing the data, which explanations are still plausible, and by how much?"

That is why the posterior matters.

It is not a prediction score. It is not a classifier output. It is your updated uncertainty after seeing the data.

And in many scientific problems, that is the real answer.
