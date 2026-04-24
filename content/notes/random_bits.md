+++
title = "Not Enough Random Bits"
date = 2026-04-23
description = "If you always seed your RNG with 0, then you always get the same \"random\" sequence. If you always seed your RNG with 0 or 1, then you can only have one of two random sequences. How many unique sequences does typical Python seeding get?"
+++

I was kinda surprised reading [What Every Experimenter Must Know
About Randomization](https://queue.acm.org/detail.cfm?id=3778029) by Terence Kelly. I never thought too hard about random seeding: as long as I was seeding, shouldn't things work out well?

Turns out, and this sounds obvious, if your seed comes from a space with lower dimension than the pseudo-random numbers you draw, then you can never reach all possible outcomes. If you only seed with 0 or 1, then there's only two possible outcomes of randomness that you could get.

I had to wonder: how many possible sequences am I getting with common non-random seeding?

# Is Now a Good Seed?

I used to think that seeding with the current time was the professional way to seed (more on that later).

```py
import random
from datetime import datetime

random.seed(datetime.now().timestamp())
```

Sure, this seed had enough precision, right? It's way better than a `random.seed(42)`, or whatever random number you happen to type in. After reading the post, I'm not so sure...

Being sloppy with leap years, a `datetime` has:
 * 1,000,000 microsecond options.
 * 60 second options.
 * 60 minute options.
 * 24 hour options.
 * 365 days options (for most years).
 * 9999 years (yes, only!).

Unfortunately, that's not a lot of options! We can check how many distinct `datetime`s we have by looking at the difference between the min and max representable times.

```py
from datetime import datetime, timedelta

print((datetime.max - datetime.min) / timedelta(microseconds=1))
```

That's a bit more than 3e17, which is a huge number!

But we should evaluate it as a power of two.

```py
from datetime import datetime, timedelta
from math import log2

print(log2((datetime.max - datetime.min) / timedelta(microseconds=1)))
```

Now that's only about 58! If we wanted our seed to cover the space of 58 random bits, then we'd need at least all of what `datetime` has to offer. Once you go beyond 58 Bernoulli draws, you can't reach every possible outcome with some seed.

To make things worse, we rarely use most of the `datetime` range. When was the last time you called `datetime.now()` and got the year to be 10? Or 3000? We're practically not reaching most possible seed values, even though the datatype allows for more.

## What Does CPython Do?

I thought `datetime` was the proper way to seed, since that's what I saw in the [Python docs](https://docs.python.org/3/library/random.html#random.seed) for `random.seed`. Without a manual seed, it checks for system randomness if possible. Otherwise, it falls back to using the timestamp.

Turns out the [CPython source](https://github.com/python/cpython/blob/main/Modules/_randommodule.c) actually does a bit more than just the timestamp; it also uses the process ID and another time to get up to 160 bits (five 32-bits) for the seed.

```c
static int
random_seed_time_pid(RandomObject *self)
{
    PyTime_t now;
    if (PyTime_Time(&now) < 0) {
        return -1;
    }

    uint32_t key[5];
    key[0] = (uint32_t)(now & 0xffffffffU);
    key[1] = (uint32_t)(now >> 32);

#if defined(MS_WINDOWS) && !defined(MS_WINDOWS_DESKTOP) && !defined(MS_WINDOWS_SYSTEM)
    key[2] = (uint32_t)GetCurrentProcessId();
#elif defined(HAVE_GETPID)
    key[2] = (uint32_t)getpid();
#else
    key[2] = 0;
#endif

    if (PyTime_Monotonic(&now) < 0) {
        return -1;
    }
    key[3] = (uint32_t)(now & 0xffffffffU);
    key[4] = (uint32_t)(now >> 32);

    init_by_array(self, key, Py_ARRAY_LENGTH(key));
    return 0;
}
```

I guess that's better than nothing, but practically calls to this function are generally done in a narrow window of times. The first two parts of the key aren't uniformly distributed over 64 bits. Same for the process ID: we're not getting zero or some other low values.

# Collisions

Let's say we had 58 truly random bits that we could use for our seed, and the randomness was uniform over the space. Essentially, the best case scenario for our seed.

We _still_ shouldn't expect to reach all possible outcomes of 58 Bernoulli draws! Two seeds could have collisions where they lead to the same first 58 bits from the pseudo random generator. If you had 58 truly random draws of 58 Bernoulli (independent, 50-50), then you should bet on a collision.

Let's go to just two bits of randomness and two draws. There are four possible seeds and four possible sequences for the two draws. There's only 24 ways for the seeds to all give different draws, but 256 possible outcomes. That's less than a ten percent chance of uniqueness! As we get more bits, the probability of uniqueness drops fast.

In some sense, this is a silly exercise. If you could get 58 truly random bits, why would you bother feeding it into a pseudo-random number generator? Just use it! We'd probably only feed it into the PRNG if we wanted _more_ than the 58 bits, at which point we're definitely not covering the full outcome space.

# True Randomness

If your system allows it, you can get arbitrarily many random bits from [`os.urandom`](https://docs.python.org/3/library/os.html#os.urandom). However, if you just use this for every number (or use [`SystemRandom`](https://docs.python.org/3/library/random.html#random.SystemRandom)) then you'll lose reproducibility, the whole point of seeding your random numbers!

You can mostly have your cake and eat it too by using the random number source for your seeds. There's many ways, since `os.urandom` just wraps any true random numbers your system may have, but since we're talking Python, you can just do it with a simple script:

```py
import os

print(os.urandom(100))
```

That script should give you 100 random bytes, which you can then use in your script. Random, but seeded.

That reminds me of a joke. Someone asks their friend to write a random number generator. The friend comes back with:
```py
def random_number():
  return 2
```
The first person says "hey, that's not random!" And the friend replies: "2 seems pretty random to me." Maybe it's not such a bad idea.
