+++
title = "It is Worth the Time"
date = "2026-03-29"
description = "Optimizing an action for speed isn't just a calculation comparing the speedup versus how many times you do the thing. That view assumes that the number of times is fixed, but speed can unlock whole new ways of working!"
[extra]
latex = true
+++

As a [fan](@/notes/polars_chunk_speed.md) [of](@/notes/uv_and_make.md) [optimizing](@/notes/rust_polars_plugin.md), I was always a bit surprised by the classic [XKCD](https://xkcd.com/1205/).

[![XKCD Is it Worth the Time?](https://imgs.xkcd.com/comics/is_it_worth_the_time.png)](https://xkcd.com/1205)

*Source: [Is It Worth the Time?](https://xkcd.com/1205) by Randall Munroe, [CC BY-NC 2.5](https://creativecommons.org/licenses/by-nc/2.5/)*

Surely those numbers are too low? Shouldn't we spend more time optimizing?

# It's Worse than That

Unfortunately, the time you should spend optimizing is _even less_ than the bleak numbers in the XKCD table! To see this, let's get into some math...

You can spend \\( t \geq 0 \\) hours optimizing a task to get the time down to \\( s(t) \\) where \\( s^\prime(t) < 0 \\). You're trying to solve a minimization problem.

\\[ \min_t s(t) + t \\]

For a nicer discussion, let's say \\( s^{\prime\prime}(t) \geq 0 \\) for some convexity, and let's assume we're not in the degenerate case where \\( s^\prime(0) \geq -1 \\).

The XKCD chart doesn't show the optimal time to spend, it shows the _breakeven_ levels where
\\[ s\left(\bar{t}\right) + \bar{t} = s(0) \\]

The optimal spend is **lower** than the breakeven, so you end up saving some time overall.

\\[ s\left(t^\ast\right) + t^\ast < s\left(\bar{t}\right) + \bar{t} = s(0)  \\]

The true optimum depends on \\( s(t) \\), which presumably varies by task and doesn't admit a nice, general-purpose chart. The chart here is really an upper bound on how much time you should spend, which is more disheartening.

# Speed and Volume: a Joint Optimization

Now for the good news: viewing speed on its own understates its benefits. The time you invest in being faster pays off in time saved later, but it also opens up new ways of working where you do it with higher frequency! A monthly task might turn into a daily task if it were faster.

In the model, let's say you're investing time \\( t \\) to speed things up, but can also just do the task more frequently. The number of times you do a task – the columns in the table – is itself an endogenous variable.

\\[ \max_{t, n} V(n) - s(t)n - t  \\]

For a fixed \\( n \\), this maximization problem reduces to the same type of time minimization problem above.

If you were going to use the XKCD chart, you'd need to know how many times you take the action. If your frequency was reasonable before the speedup, you'd use the optimal number _conditional_ on not optimizing.
\\[ n_0 = \argmax_n V(n) - s(0)n   \\]

What you actually want is the optimal number _conditional_ on optimal time spent optimizing! Well, these are really jointly determined.
\\[ (t^\ast, n^\ast) = \argmax_{t, n} V(n) - s(t)n - t   \\]

If you would do the task more if it were faster, then \\( n^\ast > n_0 \\). From the XKCD, you can see that if you'd do the task more frequently, you would be able to spend more time on optimization, which is of course consistent with the model's optimization.

\\[ s^\prime\left(t^\ast\right) = -\frac{1}{n^\ast} > -\frac{1}{n_0} = s^\prime\left(t_0\right) \\]

Where \\( t_0 \\) is the optimal spend conditional on \\( n_0 \\). Since \\( s^\prime \\) is increasing, \\( t^\ast > t_0 \\), and you should spend more time optimizing speed.

# What this Means

The somewhat-abstract math just means that speed unlocks new ways of working. For example, when a calculation is faster, you can optimize over it.

Let's say you have a simulation predicting causal outcomes for a given policy that's updated weekly. Now simulation models can be quite slow to run, at least ones with many components and policy dimensions.

At the current speeds, you only run it once a week to predict what happens with this week's choice. If you follow the chart, you shouldn't spend more than four hours optimizing it to shave off a minute. That is, assuming you only run it once a week!

However, if you can speed it up enough, you can use it to inform the weekly choice in the first place. You can run a fast simulation many times for different options and see which leads to the best outcome. The more options you evaluate, the more time should be spent optimizing speed. What once looked like a weekly task could be run the equivalent of once an hour, depending on the size of your optimization space.

It's these big step changes in operations that can really justify investing in speed.
