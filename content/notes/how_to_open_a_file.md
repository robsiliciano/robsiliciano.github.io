+++
title = "How to Open a File"
date = 2026-06-23
description = "Opening a file is one of the first things people learn when learning a language. Well, we learn the mechanics of opening a file in the standard library, but not the proper way to do it. If you're not careful, you'll end up with race condition bugs!"
+++

I was reading an overview of [security vulnerabilities in a package](https://corrode.dev/blog/bugs-rust-wont-catch/#don-t-trust-a-path-across-two-syscalls) and was surprised that the number one issue was using a path twice. If you do use the same path twice, you can run into race conditions where something you checked or set about the path changes between the calls. For example, there's the time-of-check to time-of-use (TOCTOU) family of bugs where the state of something you checked changes before you use it.

These bugs can be so tricky because you'll never run into them in single-process development. They'll only show up if there are multiple processes running at the same time, and even then, only if they happen to touch the same files at precisely the same time. How hard to reproduce!

I'd say it's better to get the best practices to avoid these bugs, rather than trying to debug them once they happen. Here are some notes on how to do everything you need in one fell swoop, closing the door on these race condition bugs.

# Setting Permissions

For a security-themed example, here's a quick script that creates a file, then sets permissions. I've added a `sleep` call to separate the two for illustrative purposes.

```py
import os
from time import sleep

if __name__ == "__main__":
    path = "test.txt"
    with open(path, "w") as f:
        f.write("Hello")

    sleep(60)

    os.chmod(path, 0o600)
```

Start that script, and while it's running, check your files with `ls -l`. You'll see `test.txt` has the default permissions of `-rw-r--r--`. Now wait for it to complete and you'll see it has the intended permissions of `-rw-------`.

That gap of time leaves the file with the wrong permissions! Try running it fresh, but this time copy `test.txt`.

```sh
cp test.txt copied.txt
```

When the script finishes and you `ls -l`, you'll see the restricted `test.txt` as intended, but an unrestricted copy `copied.txt` with the default, more permissive mode.

Now, I did make this artificially easy to copy by adding the `sleep` in the program. No one would do that on purpose, although they may have some computation between system calls (as Python implicitly does with its overhead). However, even without any gap, there's still a chance some other process accesses the files in between, leading to a permissions issue. I highlighted a long gap so we could reliably change state by hand, but the actual problem is more likely to be rare and hard to debug or catch.

## Setting Permissions Correctly

The safe way to create a file while setting permissions is to do both in one action. No gap for things to go wrong. In Python, [`open`](https://docs.python.org/3/library/functions.html#open) lets you pass an "opener" to do more specific work. In our case, we can pass a modified version of the usual opener, [`os.open`](https://docs.python.org/3/library/os.html#os.open) that selects our mode.

```py
import os
from functools import partial
from time import sleep

if __name__ == "__main__":
    path = "test.txt"
    with open(path, "w", opener=partial(os.open, mode=0o600)) as f:
        f.write("Hello")

    sleep(60)

    print("Nothing left to do!")
```

Now, the permissions are set when the file is open, so you can write to it without any risk of the file being too accessible. If you check the file while the script sleeps, you'll see the correct permissions!

## Setting Permissions in Rust

As a quick note, Rust has the helpful [`OpenOptions`](https://doc.rust-lang.org/std/fs/struct.OpenOptions.html) that lets you build an operation. You've probably used this code without knowing it: the [`File::create`](https://doc.rust-lang.org/std/fs/struct.File.html#method.create) method is just a single line call to `OpenOptions`.

# Checking Existence

Generally, don't check whether a file exists before doing something with it; that's the definition of a TOCTOU bug!

For a statistical analysis and data reconfiguration example, here's a case where we want a dataframe, preferring a cached file if it exists to manually recreating it, which might be expensive.

```py
import os
from time import sleep

import pandas as pd

if __name__ == "__main__":
    path = "test.csv"

    if os.path.exists(path):
        sleep(60)
        df = pd.read_csv(path)
    else:
        df = pd.DataFrame([])
```

The `sleep(60)` tells you where the problem will be: if the file gets deleted between `os.path.exists` and `pd.read_csv`, the program will crash.

## File Not Found

Instead, you can chain these two path operations into a single one by trying to open the file, then handling the `FileNotFoundError` if it doesn't exist.

```py
import pandas as pd

if __name__ == "__main__":
    path = "test.csv"

    try:
        df = pd.read_csv(path)
    except FileNotFoundError:
        df = pd.DataFrame([])
```

# Creating Files

For another example, let's say you need to create a set of files, but they're expensive to create. You might start with a simple script that runs through the files, finds the ones that don't exist yet, and creates them.

```py
import os
from time import sleep

import pandas as pd

def expensive_function(n) -> pd.DataFrame:
    sleep(60)
    return pd.DataFrame([n])

if __name__ == "__main__":
    for n in range(1_000):
        path = f"file_{n}.csv"
        if not os.path.exists(path):
            expensive_function(n).to_csv(path)
```

That script will get the job done, but you might be tempted to run multiple of these scripts at once. You've got all these cores to use, so why not use them to get through this twice as fast?

Well, if you were to execute a second process from this script, you wouldn't see things go any faster. Each process checks `file_0.csv`, sees it isn't there, does the expensive operation, and writes the file.

## File Exists

Just like in the past example, you can avoid this problem by doing one operation, then handling errors. In this case, the error is if the file exists, or `FileExistsError`.

```py
import os
from time import sleep

import pandas as pd

def expensive_function(n) -> pd.DataFrame:
    sleep(60)
    return pd.DataFrame([n])

if __name__ == "__main__":
    for n in range(1_000):
        path = f"file_{n}.csv"
        try:
            f = open(path, "x")
        except FileExistsError:
            continue

        try:
            with f:
                expensive_function(n).to_csv(f)
        except:
            os.remove(path)
            raise
```

Note that we want to do the operation only if the file doesn't exist, which means we have to keep the file descriptor around for the course of `expensive_function`. If we were to close it, then we'd have to reopen it again, at which point it may have been created.

Keeping it open means that we also have to handle any errors that happen in `expensive_function`. If that were to raise something, the script would exit with a blank file. Afterwards, we wouldn't know if that was the intended output or an error.

You might notice that the exception handling is using `path` twice! Once to open `f` and another time to delete the blank file if an exception occurred. We'd be fine in this example because the blank file means that no other process running our script is going to try to create it in the meantime.

As a final note, there are better ways to build this multi-process file-generating script if you're starting with that intention. However, it's easy to accidentally stumble into the race conditions too, so the best practices can still help.
