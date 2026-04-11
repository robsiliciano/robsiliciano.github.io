+++
title = "(Unit) Testing your (Frequentist) Tests"
date = 2026-04-11 
description = "I'm a big fan of unit tests for self-contained and math-heavy functions, with statistics code being a big example. If you write code to calculate pvalues or run a hypothesis test for a novel setting, you should also test it. Here are some tips on writing those inherently flakey tests."
[extra]
latex = true
+++

There's a whole suite of frequentist software – calculating pvalues, power calculations, and hypothesis tests – that should have unit test coverage. They're self-contained functions whose implementation math can be fertile ground for bugs.

However, the statistical properties of this code are inherently _flakey_! Take a method implementing some hypothesis test. You'd want to test that, conditional on the null hypothesis, the test only has a false positive 5% of the time (or \\( \alpha \\) in general). You'd test this with... Monte Carlo simulations and a statistical test, which again can fail with a small probability.

With a bit of care, we can get these flakey tests mostly under control! Here are some tips for testing your tests.

# A Basic Example

Let's start with the simplest possible test: a hypothesis test on a single value when the null is a standard normal distribution \\( \mathcal{N}(0, 1) \\). You might have code to calculate the pvalue and code to do a hypothesis test.

```python
from scipy.stats import norm

def pvalue(x: float) -> float:
    return 2 * norm.cdf(- abs(x))

def hypothesis_test(x: float, alpha: float = 0.05) -> bool:
    return pvalue(x) < alpha
```

If this works, then data from the null should be rejected by `hypothesis_test` 5% of the time. So how do we test that?

A simple option would be to draw from the null, here a standard normal, and see if it passes the hypothesis test.

```python
def test_hypothesis_test():
    x = norm.rvs()
    assert not hypothesis_test(x)
```

There are two big problems here:

First, run pytest a couple times and you'll see that this test is flakey! It fails exactly one in twenty times, which is way more than you'd like when working on a project. Imagine your tests fail 5% of the time on unrelated changes. That's not workable.

Second, we're also not fully testing what our function does. The null should usually fail to be rejected, but it should be rejected exactly 5% of the time! How can we tell that it doesn't always fail to reject the null? Or rejects 8% of the time? This little test can't do that yet.

# Seed Everything

The number one rule of these tests is to **always seed your random number generators** in each test.

In some sense, these tests are only partially flakey: if you have the same random draws and the same code, then you'll get the same results each time you run it. While not ideal, that level of flakiness is way better than flakey tests where you could get different results every time you run it.

If you seed all your random number generators, then you only have to worry about a false positive test failure when you change your code.

For instance, code that uses Python's standard [pseudo-random numbers](https://docs.python.org/3/library/random.html) and a [NumPy generator](https://numpy.org/devdocs/reference/random/index.html) would want the following in each test.

```py
import random
import numpy as np

random.seed(0)
rng = np.random.default_rng(0)
```

Note that you'd want the `random.seed` in _each_ test, not once at the top of the file. Otherwise the random state could change for each test if earlier tests have changes that leave the global RNG in a different state.

In our case, we're getting the random number from SciPy, so we'll pass in an integer `random_state`, which uses [NumPy's `RandomState`](https://numpy.org/doc/stable/reference/random/legacy.html#numpy.random.RandomState) under the hood. If you had a NumPy RNG, you could use it here too.

```python
def test_hypothesis_test():
    x = norm.rvs(random_state=0)
    assert not hypothesis_test(x)
```

Now, the test will still fail with probability 5%, but now the behavior is consistent on each run: it will either always fail or never fail. You can safely work on other changes to your codebase without worrying about this test failing.

I've used zero for the seeds here, but you should use different seeds for each test.

# Monte Carlo Testing

Now that you've got the seed taken care of, let's get at that 5% failure rate through Monte Carlo simulation.

Rather than testing one time, we need to do it a bunch and see the general average.

```python
import numpy as np

def test_hypothesis_test(N: int = 100, eps: float = 1e-3):
    rng = np.random.default_rng(0)
    results = []
    for _ in range(N):
        x = norm.rvs(random_state=rng)
        results.append(not hypothesis_test(x))

    assert np.abs(np.mean(results) - 0.95) < eps
```

The expectation should be exactly 0.95, but we only get an approximation of it through Monte Carlo integration. If we tested for equality, we'd usually fail (and always fail if \\( N \mod 20 \neq 0 \\)).

Even with the approximate equality, we'll still fail sometimes. Note that the sum of `results`, when correctly implemented, follows a binomial distribution, \\( B(N, 0.95) \\). We can exactly calculate the failure chance for a properly implemented `hypothesis test`.

```python
from scipy.stats import binom

def failure_chance(N: int, eps: float) -> float:
    return (
        # Probability of not enough failures
        1 - binom.cdf(np.ceil(N * (0.95 + eps)) - 1, N, 0.95)
        # Probability of too many failures
        + binom.cdf(N * (0.95 - eps), N, 0.95)
    )
```

For our example of `N = 100` and `eps = 1e-3`, the correct code would fail this unit test over 80% of the time! (That happens to be the probability of getting anything other than five failures, since `eps` is less than `1/N`.)

We have to get this down, and have two options: increase the tolerance `eps` or increase the number of draws `N`.

We could get the unit test failure rate down by raising the tolerance `eps = 0.06`. But do we want to? Now, we couldn't detect the difference between a correctly-implemented `hypothesis_test` and an incorrect one that just never rejects the null. We probably want our unit test to be able to capture that difference.

Instead, we have to push up the number of samples. For our `eps=1e-3`, we can get a very low error rate with... one million samples!

## Accuracy vs Speed

The more samples you have in your Monte Carlo unit tests, the more accuracy you get, but the slower they are. On one hand, you don't want some of your tests to be slow, since they have to run every time you run your full test suite, regardless of whether or not you were working on the hypothesis testing. On the other hand, you want to have a highly accurate test that can distinguish between slight errors in the rejection probability.

You can have your cake and eat it too with [parameterized tests](https://docs.pytest.org/en/stable/example/parametrize.html) and the [slow mark](https://docs.pytest.org/en/6.2.x/mark.html). Here's how you would use them to give your unit test both a fast option and an accurate option.

```python
@pytest.mark.parametrize(
    "N,eps",
    [
        pytest.param(100, 0.05),
        pytest.param(1_000_000, 1e-3, marks=pytest.mark.slow),
    ],
)
def test_hypothesis_test(N, eps):
    ...
```

When you run `pytest`, it runs both tests: the fast one (`N=100`, `eps=0.05`) and the accurate one (`N=1_000_000`, `eps=1e-3`). You'll notice the first one is quick but the second one takes forever.

If you just want to run the fast tests, you can tell `pytest` to avoid slow tests.
```sh
pytest -m 'not slow'
```

You can also configure pytest to leave out your slow, high-sample tests for default runs, if that makes sense for your use case.

# Testing pvalues

We've already done a basic unit test of the pvalue function with the hypothesis test unit test, but we can do better. Since we're testing the pvalue of a continuous distribution, the pvalues should be distributed uniformly (though this isn't true for discrete distributions, but we'll handle that case too). We can just repeat the same logic of the hypothesis test, reusing the parameterization this time to run on many values.

```python
@pytest.mark.parametrize(
    "alpha",
    [0.0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9],
)
def test_pvalue(alpha):
    rng = np.random.default_rng(0)
    results = []
    for _ in range(100):
        x = norm.rvs(random_state=rng)
        results.append(pvalue(x) < alpha)

    assert np.abs(np.mean(results) - alpha) < 0.05
```

This test suite would be inefficient and really push on our accuracy vs speed tradeoff: we're running ten Monte Carlo simulations when we could just be running one. For example, we could use something like a chi-squared test on the results.

```python
from scipy.stats import chisquare

def test_pvalue_chisqr(N: int = 100, nbins: int = 10):
    # Monte Carlo to approximate the pvalue distribution
    rng = np.random.default_rng(0)
    pvalues = []
    for _ in range(N):
        x = norm.rvs(random_state=rng)
        pvalues.append(pvalue(x))

    # Put the distribution into bins
    bins = np.linspace(0, 1, nbins + 1)
    counts, _ = np.histogram(pvalues, bins=bins)

    _, chisqr_pvalue = chisquare(counts, [N/nbins] * nbins)
    assert chisqr_pvalue > 0.01
```

The chi squared works when we might not have a continuous distribution, where pvalue distribution could be not exactly uniform. You wouldn't have a pvalue distributed \\( U[0, 1] \\) with a discrete distribution like the Binomial. However, we're working with a uniform distribution here, so we can also use a [KS test](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.kstest.html) in our unit test.

```python
from scipy.stats import kstest, uniform

def test_pvalue_ks(N: int = 100):
    # Monte Carlo to approximate the pvalue distribution
    rng = np.random.default_rng(0)
    pvalues = []
    for _ in range(N):
        x = norm.rvs(random_state=rng)
        pvalues.append(pvalue(x))

    ks = kstest(pvalues, uniform.cdf)
    assert ks.pvalue > 0.01
```

In either case, you can make the pvalue threshold smaller to reduce the chance of a false unit test failure. You'll need to increase `N` to offset the lower threshold, again using the parameterization if the tests are too slow.

# Other Things to Test

1. **Power calculations**. You would test your power calculations much like the unit tests for hypothesis testing, but in reverse. If you're estimating the sample size required, draw data with that sample size many times, and ensure you reject the null the right amount of times.

2. **Confidence intervals**. Under different true parameter values, the confidence interval should contain the true value with the proper probability. Again, use Monte Carlo simulation to draw and statistical tests to check that the probabilities are correct.
