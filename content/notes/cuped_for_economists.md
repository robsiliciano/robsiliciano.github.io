+++
title = "CUPED for Economists"
date = 2026-03-22
description = "CUPED, a core part of causal inference at tech companies, sounds unfamiliar to economists at first, but is really just a basic regression concept. Here's a quick introduction to what CUPED is, how it boils down to regressions, and what's unique about CUPED if not the math."
[extra]
latex = true
+++

I think I had the typical experience of an economist entering a large tech firm and being shocked at the power of their experimentation platforms. I had spent years studying complicated causal inference that could work in quasi-experimental settings, but the ability to randomize at scale let the companies run on just simple A/B tests to get good causal estimates. The statistics powering these companies is basically the same difference in means that I used in high school biology!

I was particularly shocked that the estimator these companies use, [CUPED](https://dl.acm.org/doi/abs/10.1145/2433396.2433413), wasn't something I had heard about in graduate school. If CUPED was such a widespread tool for causal inference in practice, why didn't we learn about it?

Turns out, we did learn about it, and CUPED is just the rebranded concept of covariates in a regression. [Evan Miller](https://www.evanmiller.org/you-cant-spell-cuped-without-frisch-waugh-lovell.html) and [Matteo Courthoud](https://matteocourthoud.github.io/post/cuped/) have great posts on this. The CUPED estimator basically just applies Frisch–Waugh–Lovell to an A/B test.

I found it helpful to understand CUPED as regression in disguise, but once I thought about that, I had to ask: why use CUPED at all? There's more to it than just the math, but before I can answer that question, we have to go through the math anyways.

# The Math of CUPED

We'd like to estimate the effect of some treatment \\(T \in \{0, 1\} \\) on some outcome \\( Y \\).

\\[ \tau = \mathbb{E}[Y \mid T = 1] - \mathbb{E}[Y \mid T = 0]  \\]

The usual A/B estimate is just the difference in means between the treatment and control group.

\\[ \tau_{\textrm{A/B}} = \frac{1}{\sum_{T_i = 1} 1} \sum_{T_i = 1} Y_i - \frac{1}{\sum_{T_i = 0} 1} \sum_{T_i = 0} Y_i \\]

We've got a fine estimator in that it's unbiased. However, you have to run a large enough experiment to be powered for small treatment effects. When you're running experiments for almost every change, the treatment effects you care about can be small, but you don't have a ton of time to wait in experimentation.

CUPED's value is to reduce the variance of the A/B estimator, which lets companies run smaller and shorter experiments.

## CUPED in a Nutshell

To quickly run through CUPED, you take pre-experiment variables, \\( X_i \\), and compute an adjusted outcome.

\\[ \tilde{Y}_i = Y_i - \hat{\theta} X_i  \\]

The factor \\( \hat{\theta} \\) is the coefficient from regressing \\( Y_i \\) on \\( X_i \\). Usually, \\( X_i \\) is the same metric as \\( Y_i \\) but measured before, rather than after, the treatment. In that case, you have the simple formula for the regression coefficient.

\\[ \theta = \frac{\textrm{Cov}(Y, X)}{\textrm{Var}(X)} \\]

Then, the CUPED estimator is just the difference in means of the adjusted outcomes.

{% katex() %}
\tau_{\textrm{CUPED}} = \frac{1}{\sum_{T_i = 1} 1} \sum_{T_i = 1} \tilde{Y}_i - \frac{1}{\sum_{T_i = 0} 1} \sum_{T_i = 0} \tilde{Y}_i
{% end %}

It turns out that this is an unbiased estimator: \\( \mathbb{E}(\tau_{\textrm{CUPED}}) = \mathbb{E}(\tau_{\textrm{A/B}}) = \tau \\). Furthermore, it has lower error, through lower variance, provided there's a nonzero correlation between \\( Y_i \\) and \\( X_i \\).

## It's Just Covariates

The typical analogue would be covariates in a linear regression.

\\[ Y_i = \alpha + \tau T_i + \theta X_i + \epsilon \\]

With the **Frisch–Waugh–Lovell theorem**, you can residualize \\( Y_i \\) and \\( T_i \\) by regressing both of them on \\( X_i \\).

{% katex() %}
\begin{align*}
  Y_i &= \alpha_Y + \theta \cdot X_i + \epsilon_Y  \\
  \tilde{Y}_i &= Y_i - \hat{\alpha}_Y - \hat{\theta} \cdot X_i  \\
  T_i &= \alpha_T + \beta_T \cdot X_i + \tilde{T}_i \\
  \tilde{T}_i &= T_i - \hat{\alpha}_T - \hat{\beta}_T \cdot X_i  \\
\end{align*}
{% end %}

Then, you can recover \\( \tau \\) easily from regressing \\( \tilde{Y}_i \\) on \\( \tilde{T}_i \\).

\\[ \tilde{Y}_i = \tau \tilde{T}_i + e  \\]

The estimate for \\( \tau \\) in the residualized equation with just two variables is the same that you'd get from the full regression.

It turns out that CUPED is almost the same as this FWL approach, with some slight shortcuts given that \\( T \\) is coming from an A/B test. The magic variance reduction is just the typical variance reduction from adding covariates to the regression. (I'd use the phrase "controls" but that's an overloaded term here.)

## Unifying the Tildes

I pulled a fast one and gave two slightly _different_ definitions for \\( \tilde{Y}_i \\), depending on the model. CUPED just subtracts \\( \hat{\theta} X_i \\).
\\[ \tilde{Y}^{\textrm{CUPED}}_i = Y_i - \hat{\theta} X_i \\]
On the other hand, linear regression also subtracts a scalar \\( \hat{\alpha}_Y \\).
\\[  \tilde{Y}_i = Y_i - \hat{\alpha}_Y - \hat{\theta} X_i = \tilde{Y}^{\textrm{CUPED}}_i - \hat{\alpha}_Y  \\]

The level change turns out not to matter for the CUPED estimator, since it would cancel between treatment and control. If you computed the CUPED estimator with the proper residuals \\( \tilde{Y}_i \\), you'd get the exact same result.
{% katex() %}
\begin{align*}
  &\frac{1}{\sum_{T_i = 1} 1} \sum_{T_i = 1} \tilde{Y}_i - \frac{1}{\sum_{T_i = 0} 1} \sum_{T_i = 0} \tilde{Y}_i \\
  &= \frac{1}{\sum_{T_i = 1} 1} \sum_{T_i = 1} (\tilde{Y}^{\textrm{CUPED}}_i - \hat{\alpha}_Y) - \frac{1}{\sum_{T_i = 0} 1} \sum_{T_i = 0} (\tilde{Y}^{\textrm{CUPED}}_i - \hat{\alpha}_Y) \\
  &= \frac{1}{\sum_{T_i = 1} 1} \sum_{T_i = 1} \tilde{Y}^{\textrm{CUPED}}_i - \frac{1}{\sum_{T_i = 1} 1} \sum_{T_i = 1} \hat{\alpha}_Y - \frac{1}{\sum_{T_i = 0} 1} \sum_{T_i = 0} \tilde{Y}^{\textrm{CUPED}}_i + \frac{1}{\sum_{T_i = 0} 1} \sum_{T_i = 0} \hat{\alpha}_Y \\
  &= \frac{1}{\sum_{T_i = 1} 1} \sum_{T_i = 1} \tilde{Y}^{\textrm{CUPED}}_i - \hat{\alpha}_Y - \frac{1}{\sum_{T_i = 0} 1} \sum_{T_i = 0} \tilde{Y}^{\textrm{CUPED}}_i + \hat{\alpha}_Y \\
  &= \frac{1}{\sum_{T_i = 1} 1} \sum_{T_i = 1} \tilde{Y}^{\textrm{CUPED}}_i - \frac{1}{\sum_{T_i = 0} 1} \sum_{T_i = 0} \tilde{Y}^{\textrm{CUPED}}_i \\
  &= \tau_{\textrm{CUPED}}
\end{align*}
{% end %}

The CUPED version is faster and simpler to calculate, but still gets you where you need to go.

## The Treatment Residual

The other main difference between the two methods is that linear regression needs to use the _residualized_ treatment, \\( \tilde{T}_i \\), while CUPED doesn't. CUPED can get away with this because we're properly randomizing \\( T \\), so \\( X_i \\) should have no correlation with it! Unlike many quasi-experimental settings where we'd have to worry about \\( \beta_T \\), in a true experiment, the value of this coefficient is zero.

To put it another way, let's start with the FWL equation and move towards CUPED.

{% katex() %}
\begin{align*}
  \tilde{Y}_i &= \tau \tilde{T}_i + e \\
  \tilde{Y}^{\textrm{CUPED}}_i - \hat{\alpha}_Y &= \tau \tilde{T}_i + e \\
  \tilde{Y}^{\textrm{CUPED}}_i &= \hat{\alpha}_Y + \tau \tilde{T}_i + e \\
  \tilde{Y}^{\textrm{CUPED}}_i &= \hat{\alpha}_Y + \tau \left(T_i - \hat{\alpha}_T - \hat{\beta}_T \cdot X_i \right) + e \\
  \tilde{Y}^{\textrm{CUPED}}_i &= \hat{\alpha}_Y - \tau \hat{\alpha}_T + \tau T_i + \left(e - \tau \hat{\beta}_T \cdot X_i \right)
\end{align*}
{% end %}

The error here has the nice asymptotics for a regression (it's consistent and unbiased) where the regression estimate would happen to be \\( \tau_{\textrm{CUPED}} \\)!

# Why Reinvent the FWheeL?

Okay, so CUPED is really a century-old result on regression, applied to A/B testing. Even if CUPED isn't novel mathematically, it clearly has value given by its widespread adoption.

I don't think CUPED shows up in academia because they lack experimentation platforms. There's no common system that all researchers use for reading experiment results. Every analysis is, to some degree, starting from scratch.

In contrast, tech companies usually have an experimentation platform that takes care of all the mechanics of experimentation and analysis. They need to both implement and measure, so there's a lot more demand for measurement tooling when it's a small part of the whole project. And for that, you have to randomize, measure, and produce results at scale.

With A/B tests as the backbone or starting point for these systems, I expect they'd usually end up with a process for presenting and comparing the means for the treatment and control groups.

If you want to move towards a regression that can include more covariates, then you'd have to rebuild a lot of that infrastructure.

CUPED offers a faster route to covariates. All you need to do is produce the \\( \tilde{Y}^{\textrm{CUPED}}_i \\) estimates, then you can pipe them into the system as if they were the experiment's outcome variable.

Overall, I think CUPED's main benefit is bringing well-known causal inference ideas into large systems in a practical way. Moving ideas and techniques to systems that scale is still valuable, even if the math has been around for a century of academic papers.
