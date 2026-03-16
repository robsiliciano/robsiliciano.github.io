+++
title = "Confidence Interval Problems"
date = 2026-03-15
description = "I dug up some of my favorite examples from Chris Sims's class of what can go wrong with confidence intervals. We all know that we can't interpret them as a statement about post-sample or posterior uncertainty, and he had some fun cases where confidence intervals were clearly wacky."
[extra]
latex = true
+++

I was rereading the [course notes](http://sims.princeton.edu/yftp/Courses/UndergradEmet13/) for a class I took with the late, great Chris Sims, and I remembered how many good examples he had where frequentist interpretations could go awry. Some of my favorite are on the misinterpretation of confidence intervals.

# Confidence Intervals

It seems like every interview for an applied/data/research science job in tech asks candidates what a confidence interval is. The correct answer is to say that a 95% confidence interval \\( (u(X), v(X) ) \\) for some parameter \\( \theta \\) is a function of some observed data \\( X \sim F(\theta) \\) where, conditional on the parameter and before drawing the data, the probability that the confidence interval includes the parameter is 95%.
\\[ \forall \theta \\quad \mathbb{P}\left( u(X) \leq \theta \leq v(X) \mid \theta \right) = 0.95 \\]

The question tries to catch a common misunderstanding of confidence intervals where people sometimes interpret them as saying something **after the data has been seen**. In a frequentist sense:
\\[ \\quad \mathbb{P}\left( u(X) \leq \theta \leq v(X) \mid X, \theta \right) = 0.95 \\]

As [Chris Sims said](http://sims.princeton.edu/yftp/Times16/BayesBasics.pdf):
> After the data has been seen, the probability is zero or one.

\\[ \\quad \mathbb{P}\left( u(X) \leq \theta \leq v(X) \mid X, \theta \right) \in \{ 0, 1 \} \\]

People also frequently confuse frequentist confidence intervals (\\( X \\) unknown, \\( \theta \\) known) with the Bayesian credible intervals (\\( X \\) known, \\( \theta \\) unknown). The latter is _much_ more useful in business, but generally not what people have calculated.

Yet despite how frequently the confidence interval question comes up in interviews for a job, I think people keep mixing them up and using confidence intervals for a measure of posterior uncertainty!

# Getting Away With It

I think people stay confused about confidence intervals because they do happen to match credible intervals in some large and common cases! You can pretend your confidence intervals tell you about posterior uncertainty and sometimes be right. I guess this is the statistics version of "a broken clock is right twice a day."

If you had to assume a simple model, you might take \\( X \mid \theta \sim \mathcal{N}(\theta, \sigma^2) \\). Add in a flat prior on \\( \theta \\) and you'll find that \\( \theta \mid X \sim \mathcal{N}(X, \sigma^2) \\). It just so happens that the confidence interval and the credible interval are the same!
\\[ \mathbb{P}\left( X - 1.96 \sigma \leq \theta \leq X + 1.96 \sigma \right) \approx 0.95  \\]
\\[ \mathbb{P}\left( \theta - 1.96 \sigma \leq X \leq \theta + 1.96 \sigma \right) \approx 0.95  \\]

Asymptotics approach this case too, where (most) distributions start to look normal and the prior has less and less influence. Unless you have a dogmatic prior, that is. If someone's comfortable making asymptotic assumptions, they can take their confidence intervals as credible intervals.

To [quote Chris Sims again](http://sims.princeton.edu/yftp/EmetSoc607/AppliedBayes.pdf),
> Good frequentist practice has a Bayesian interpretation
> This is a definition, not a theorem.

People get away with misinterpreting confidence intervals as credible intervals because they usually made so many assumptions that the two intervals match.

# Silly Confidence Intervals

Okay, so there's one big case where misinterpretation works fine, but anywhere else you could have problems.

_I've adapted this from [the first lecture in the class](http://sims.princeton.edu/yftp/Courses/UndergradEmet13/ProbAndInf.pdf)._

Let's say you have a binomial \\( X \sim B(n, p) \\). As \\( n \to \infty \\), you can use the usual normal approximations for your confidence intervals and have them match with the credible intervals.

However, let's take the opposite side and look for small \\( n \\). When you get \\( X = 0 \\) or \\( X = n \\), what do you do? How do you handle a confidence interval of \\( [0, 0] \\) or \\( [1, 1] \\)? You could interpret those confidence intervals as saying you had a very accurate estimate, but that would be wrong. For any \\( p \in (0, 1) \\), if \\( n = 1 \\) then you're guaranteed to get \\( X \in \{ 0, 1\} \\) and one of those weird confidence intervals. Surely you wouldn't believe that a Bernoulli can only have probabilities either zero or one.

## Bad Intervals

You can have goofy confidence intervals like \\( [0, 1] \\) because confidence intervals are a concept _before_ the data is realized.

Let's say your data is \\( (X, Y) \\) where you have \\( Y \sim U[0, 1] \\) unrelated to the parameter of interest. You _could_ create a terrible confidence interval:
* if \\( y > 0.95 \\) then pick a random \\( [p, p] \\).
* if \\( y \leq 0.95 \\) then use the full parameter space \\( [0, 1] \\).

Generically, this interval is a 95% confidence interval because 95% of the time it's so wide that it captures everything and the remaining 5% of the time it's so small it captures almost nothing. This example is contrived and a bit evil, but it's worth remembering that presample and postsample can mean very different things.

# Asymmetric Problems

Here's another of his fun examples: \\( X = \beta + \epsilon \\) where \\( f(\epsilon) \propto e^{-\epsilon} \\) on \\( \epsilon \in [0, \infty) \\). This distribution is entirely skewed, with \\( \mathbb{P}(X > \beta) = 1 \\). There's no support for \\( X < \beta \\).

## MLE? Maybe Not.

I think most people might reach for a maximum likelihood estimator when faced with a weird distribution, but that can be dangerous. In this case, the maximum likelihood estimator is \\( \hat{\beta} = X \\), a clearly upward biased estimator.

You'd want to do \\( X - \mathbb{E}(\epsilon) \\) instead, which requires thinking carefully about the model rather than applying a general framework. In contrast, a Bayesian has a nice time since you can use the same approach for this model as you would for the nice normal ones. 

## Bootstrapped CIs? Maybe Not.

With such asymmetry, the common approach of bootstrapping your confidence interval will also get you into trouble. The parameter is on the bounds of the support for \\( X \\), with no point mass. You'll end up with a confidence interval strictly larger than the point estimate, and it will _never_ contain the true parameter!

All this to remember that frequentist tools are really context dependent, and if you're not careful, you might use a well-respected technique to end up with something clearly wrong.

# What to Read

Anyways, Chris Sims put a lot of his course materials and notes on [his website](https://www.princeton.edu/~sims/).
