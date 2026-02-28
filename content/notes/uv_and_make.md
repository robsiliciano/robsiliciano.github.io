+++
title = "Make-ing with uv"
date = 2026-02-28
+++

I love using Makefiles to organize my statistical analysis and data reconfiguration projects, but I always found it painful to properly write rules for my virtual environments. I'd usually risk it and leave dependency management out. Hopefully there wouldn't be a package change that could affect any of my rules!

I recently wondered if `uv` made package management so easy that I didn't have to worry about virtual environments in Makefiles any more! Short answer: no, but it does make things easy enough that you can put your package versions into your dependency graph. Here's a reasonably short adjustment to your Makefiles to get a bit of added peace of mind.

# The Problem

Let's say you've got a simple project with two steps. A first script, `script1.py` makes `file1.parquet`, then a second script, `script2.py`, makes `file2.png`.

```Makefile
file2.png: script2.py file1.parquet
	uv run script2.py file1.parquet

file1.parquet: script1.py
	uv run script1.py
```

Running `make` will appropriately build dirty targets. It will rebuild everything when `script1.py` changes, but it will leave `file1.parquet` alone when only `script2.py` changes.

However, you've got a problem: what if you add or change a dependency and it affects the functionality of either script? `uv` takes care of everything, but _only_ when you `uv run` a script. It won't help on skipped steps!

## A First Pass

If you want to force all the targets to update whenever you change dependencies, you can add `pyproject.toml` to all your rules.

```Makefile
file2.png: script2.py file1.parquet pyproject.toml
	uv run script2.py file1.parquet

file1.parquet: script1.py pyproject.toml
	uv run script1.py
```

This Makefile is _fine_ but we can improve it a bit.

# Rules on Rules

I don't love having all those extra `pyproject.toml`s cluttering up my rules: it makes the dependencies harder to read with long lines and line wraps.

Fortunately, `make` supports [multiple targets](https://www.gnu.org/software/make/manual/html_node/Multiple-Targets.html) where we can define the extra dependencies on a separate line.

```Makefile
target1:
	echo target1

target2:
	echo target2

target1: target2
```

In that silly example, we didn't simplify anything by having a second `target1` rule. The pattern only starts to pay off once we need to add the additional dependency to several other targets. Then, we can do them all with one rule!

For the project example, we could alternatively do:

```Makefile
file2.png: script2.py file1.parquet
	uv run script2.py file1.parquet

file1.parquet: script1.py
	uv run script1.py
  
file2.png file1.parquet: pyproject.toml
```

Or introduce a variable if things get long:

```Makefile
file2.png: script2.py file1.parquet
	uv run script2.py file1.parquet

file1.parquet: script1.py
	uv run script1.py
  
PYTHON_TARGETS = file2.png file1.parquet
$(PYTHON_TARGETS): pyproject.toml
```

## Pattern Rules

Why not a pattern rule though? The blank `%` pattern could match everything.

```Makefile
%: pyproject.toml
```

Unfortunately, this pattern rule will never match, since `make` only checks for patterns if it can't find a rule that matches. You can only append additional targets with explicit rules.

## Extra Prereqs

On newer versions of GNU `make`, since circa 2020, there's also the `.EXTRA_PREREQS` [special variable](https://www.gnu.org/software/make/manual/html_node/Special-Variables.html), which lets you set additional dependencies for every rule by default. You can turn them off on a rule-by-rule basis though.

```Makefile
target1:
	echo target1

target2:
	echo target2

.EXTRA_PREREQS: target2
```

Unfortunately, not everyone has `make` from the last five years. One system (cough, cough Macs) hasn't updated GNU `make` since the project switched from [GPL 2 to GPL 3 in 2007](https://cgit.git.savannah.gnu.org/cgit/make.git/commit/?id=891ebd4d9766c1fb0bd11bd0fe8ef3ca871d4bc0). That's almost 20 years old now! Maybe it's time to update your software?

```Makefile
file2.png: script2.py file1.parquet
	uv run script2.py file1.parquet

file1.parquet: script1.py
	uv run script1.py

.EXTRA_PREREQS: pyproject.toml
```

# Lockfiles

Eventually, you'll notice that everything rebuilds a few too many times. If you edit `pyproject.toml`, then the current Makefile says that everything that uses Python needs to be redone. Change the version? Rerun. Change the description? Rerun. Those cases are too aggressive!

We only cared about `pyproject.toml` because it contains the package environment, which could invalidate the rebuild. Instead, we should care about the lockfile, `uv.lock`, which just contains package information. Normally, `uv` would take care of rebuilding the `uv.lock` every time we do a `uv run`, but we can force it in a specific rule with `uv sync`.

```Makefile
file2.png: script2.py file1.parquet
	uv run script2.py file1.parquet

file1.parquet: script1.py
	uv run script1.py
  
PYTHON_TARGETS = file2.png file1.parquet
$(PYTHON_TARGETS): uv.lock

uv.lock: pyproject.toml
	uv sync
```

Unfortunately, this _still_ rebuilds on unrelated changes to the `pyproject.toml`. `make` works off timestamp, so doesn't know if `uv sync` made any changes or not. As long as `uv.lock` was touched, it has to rerun all Python targets.

Let's _only_ rebuild when the environment changes using `uv lock --check` to identify if the lockfile needs to change. 

```Makefile
uv.lock: pyproject.toml
	@uv lock --check --quiet || uv sync
```

Now you should be able to change version information without a full rebuild!

# The Final Product

Altogether, the Makefile would look like:

```Makefile
file2.png: script2.py file1.parquet
	uv run script2.py file1.parquet

file1.parquet: script1.py
	uv run script1.py
  
PYTHON_TARGETS = file2.png file1.parquet
$(PYTHON_TARGETS): uv.lock

uv.lock: pyproject.toml
	@uv lock --check --quiet || uv sync
```

Whether you use `.EXTRA_PREREQS` or not, you still have to manually tell `make` which targets should depend on `uv.lock`. There are messy ways to do this by printing `make`'s internal DB for the Makefile, but that's a bit complexity than I'd like. It's easy enough for me to write the `PYTHON_TARGETS` variable, but feel free to try out other options.
