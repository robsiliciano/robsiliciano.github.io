+++
title = "Pip is Chaos"
date = 2026-01-30
+++

Nothing in Python scares me like `pip`, the included package manager. Anytime I see `pip` written or used, I immediately worry about an upcoming dependency hell!

Shockingly, `pip` doesn't guarantee your Python environment has the right versions of all the dependencies needed for your packages. Isn't that a basic requirement for a package manager? `pip` _usually_ does, but there's some cases when it just won't.

# A Dependency-Conflict Example

To show you, let's set up an example of a dependency conflict with three packages: A, B, and C. We'll want to install A and B, but they depend on different, incompatible versions of C! `pip` is going to sometimes let us end up with all three packages installed.

```sh
uv init --lib a
uv init --lib b
uv init --lib c
```

I'm using `uv` to make packaging simple (though it'll turn out to be so much more useful at the end).

## An Index of Your Own
 
I don't want to put any of this in PyPi, but we can replace the default with our own [local index](https://packaging.python.org/en/latest/guides/hosting-your-own-index/), right on our filesystem! I'm going to make a barely [PEP-503 compatible](https://packaging.python.org/en/latest/specifications/simple-repository-api) one, which is basically just folders and `index.html` files pointing at wheels and source distributions.

Let's make an index directory to hold this index.

```sh
mkdir index
```

First, we'll need an `index.html` file to list all the packages in this index.

```html,name=index/index.html
<!doctype html>
<html>
    <body>
        <a href="a/">a</a>
        <a href="b/">b</a>
        <a href="c/">c</a>
    </body>
</html>
```

## Package C

I'll start at the end, with the shared dependency. We'll want to make two versions of this package, then add both to the index.

First, let's [build](https://docs.astral.sh/uv/guides/package/#building-your-package) C 0.1.0.
```sh
uv build
```
 
You should see a `dist` directory with `c-0.1.0.tar.gz` and a wheel file (`c-0.1.0-???.whl`) in it.

Let's go make C 0.2.0. A [quick way](https://docs.astral.sh/uv/guides/package/#updating-your-version) is:
```sh
uv version --bump minor
```
 
Now you should see that updated version in the configuration.
```toml,name=C/pyproject.toml
version = "0.2.0"
```

Build with `uv build` again and you should have a `c-0.2.0.tar.gz` and corresponding wheel in the `dist` directory, along side the 0.1.0 version.

Now, to add these C versions to your index, make a `index/c` directory and add the two wheels.

```html,name=index/c/index.html
<!doctype html>
<html>
    <body>
        <a href="../../c/dist/c-0.1.0-py3-none-any.whl">
            c-0.1.0-py3-none-any.whl
        </a>
        <a href="../../c/dist/c-0.2.0-py3-none-any.whl">
            c-0.2.0-py3-none-any.whl
        </a>
    </body>
</html>
```

We want wheels and not source distributions (`.tar.gz`) so we won't have to build them. That building step would require build tools, which is `uv-build` in our case. If you wanted to go that route, you'd have to add a `uv-build` wheel to the index as well, linking to some file from [PyPI](https://pypi.org/simple/uv-build).

## Package A

Now, let's build the first package that requires C. Package A will require version 0.1.0, but not be compatible with 0.2.0.
```sh
cd ../A
```

You'll need to configure `uv` to point towards the your index. Replace the url with the _absolute_ path to your index. Note that you should have three slashes to start the file path, the first two follow `file:` and the third is the start of the absolute path.

```toml,name=a/pyproject.toml
[[tool.uv.index]]
name = "local"
url = "file:///???/index"
default = true
```

Add C version 0.1.0 as a dependency with `uv`
```sh
uv add "c==0.1.0"
```

You should see `c` in A's `pyproject.toml` with the appropriate version.

Now build A. You should see a `dist` directory with `a-0.1.0.tar.gz` and a wheel in it.
 
```sh
uv build
```

Finally, add A to the index, again using the wheel you just built.

```html,name=index/A/index.html
<!doctype html>
<html>
    <body>
        <a href="../../a/dist/a-0.1.0-py3-none-any.whl">
            a-0.1.0-py3-none-any.whl
        </a>
    </body>
</html>
```

## Package B

We'll repeat the same steps for B, but now B will require version 0.2.0 and not be compatible with 0.1.0.
```sh
cd ../B
```

Add the same index configuration, again replacing the url with the absolute, not relative, path to the index.
```toml,name=B/pyproject.toml
[[tool.uv.index]]
name = "local"
url = "file:///???/index"
default = true
```

Add C version _0.2.0_ with
```sh
uv add "c==0.2.0"
```

Build B. You should see a `dist` directory with `b-0.1.0.tar.gz` and the wheel in it.
 
```sh
uv build
```

Add B to the index.

```html,name=index/B/index.html
<!doctype html>
<html>
    <body>
        <a href="../../b/dist/b-0.1.0-py3-none-any.whl">
            b-0.1.0-py3-none-any.whl
        </a>
    </body>
</html>
```

## Installing Conflicts

Now, let's see `pip` mess up with this dependency conflict! Set up a virtual environment (which is usually the first step to dependencies going wrong).

```sh
cd ..
mkdir conflict
cd conflict
python3 -m venv .venv
source .venv/bin/activate
```

Now try to add A and B at the same time, which we know to be impossible. The `--index-url` will tell `pip` to use our own index.

```sh
pip install --index-url file://$(PWD)/../index a b
```

If everything went according to plan, the installation should fail. We asked `pip` to do something impossible, so it shouldn't do anything.

```
INFO: pip is looking at multiple versions of b to determine which version is compatible with other requirements. This could take a while.
ERROR: Cannot install a==0.1.0 and b==0.1.0 because these package versions have conflicting dependencies.

The conflict is caused by:
    a 0.1.0 depends on c==0.1.0
    b 0.1.0 depends on c==0.2.0

Additionally, some packages in these conflicts have no matching distributions available for your environment:
    c
```

## Sequential Installs

The big danger with `pip` comes from installing one package at a time, which people do all the time.

```sh
pip install --index-url file://$PWD/../index a
pip install --index-url file://$PWD/../index b
```

The first command will successfully install A and C version 0.1.0, which we expect. The problem comes in the second step, which _uninstalls_ C then reinstalls 0.2.0. `pip` knows that A doesn't like this newer version, but goes ahead anyways.

```
Installing collected packages: c, b
  Attempting uninstall: c
    Found existing installation: c 0.1.0
    Uninstalling c-0.1.0:
      Successfully uninstalled c-0.1.0
ERROR: pip's dependency resolver does not currently take into account all the packages that are installed. This behaviour is the source of the following dependency conflicts.
a 0.1.0 requires c==0.1.0, but you have c 0.2.0 which is incompatible.
Successfully installed b-0.1.0 c-0.2.0
```

The resulting Python environment is inconsistent! List packages to see this.

```sh
pip freeze
```

```
a==0.1.0
b==0.1.0
c==0.2.0
```

# Capping Dependencies

The whole inconsistency here came down to A not accepting a new version for C. Had A instead allowed any new version of C (`c>=0.1.0`) then `pip` would have been able to install a consistent environment.

Don't rush to uncap all dependencies though! There's a good reason to cap, especially for major versions. In [semantic versioning](https://semver.org), major version bumps can have API-breaking changes. If your dependency breaks their API, then your project will start failing with no change to the code (and only in new installations)!

This error happened in the real world when NumPy released 2.0.0. Anyone in [Statistical analysis and data reconfiguration](https://www.youtube.com/watch?v=fXpVEwEhamw) uses NumPy and pandas together on a daily basis, frequently starting off with a `pip install pandas` to get both packages. All of a sudden, code started failing as pandas [didn't cap NumPy](https://github.com/pandas-dev/pandas/issues/59023) but didn't handle the new API changes. It took [until pandas 2.2.2](https://pandas.pydata.org/docs/whatsnew/v2.2.2.html#pandas-2-2-2-is-now-compatible-with-numpy-2-0) for compatibility.

# What About Requirements?

Sometimes people will recommend using a [`requirements.txt` file](https://pip.pypa.io/en/stable/reference/requirements-file-format/) to define the entire environment in a reproducable way. These files contain a long list of packages and versions that `pip` will evaluate all at once, sidestepping the one-by-one installation.

I've usually seen people use their current Python environment to construct the `requirement.txt` file by listing every package and its version. These people assume that listing everything in their environment is enough to reconstruct it elsewhere.

```sh
pip freeze > requirements.txt
```

Unfortunately, there's no guarantee that the `requirements.txt` is even installable! If their initial environment was inconsistent, then `pip` will fail to install the `requirements.txt` at all. In our case, we'd fail. Running that installation:

```sh
pip install --index-url file://$(PWD)/../index -r requirements.txt
```

Would fail with the same impossible resolution error.
```
ERROR: Cannot install a==0.1.0 and c==0.2.0 because these package versions have conflicting dependencies.

The conflict is caused by:
    The user requested c==0.2.0
    a 0.1.0 depends on c==0.1.0

Additionally, some packages in these conflicts have no matching distributions available for your environment:
    c
```

A `requirements.txt` is not a robust lockfile. It may miss dependencies or include inconsistencies. We can avoid the first issue of missing dependencies with `pip freeze`, but we won't know that those dependencies are all compatible.

## Compiling

If you _absolutely must_ use a `requirements.txt` file, please compile it from a `pyproject.toml` or even a `requirements.in` file. Either use the original `pip-compile` from [`pip-tools`](https://pip-tools.readthedocs.io/en/stable/) or use [`uv pip compile`](https://docs.astral.sh/uv/pip/compile/).

You could create a `requirements.in` file with just the direct packages that you want.

```name=requirements.in
a==0.1.0
b==0.1.0
```

Then, you can compile it into a `requirements.txt`.

```sh
uv pip compile --index-url file://$(PWD)/../index requirements.in -o requirements.txt
```

If your requirements were consistent, then compilation would produce a consistent and complete `requirements.txt`. However, it will fail in this case (which we want!) because ther requirements are inconsistent, and hence impossible to satisfy.
```
× No solution found when resolving dependencies:
╰─▶ Because a==0.1.0 depends on c==0.1.0 and b==0.1.0 depends on c==0.2.0, we can conclude that a==0.1.0 and b==0.1.0 are incompatible.
    And because you require a==0.1.0 and b==0.1.0, we can conclude that your requirements are unsatisfiable.
```

# Don't Use Pip

The only way to safely and reproducibly manage Python environments is to not manage Python environments, at least not directly!

In practice, everyone sometimes needs to add or remove packages from an environment. Unfortunately, sequentially adding them with `pip` is a great way to mess things up. Once you wind up with an inconsistent environment or `requirements.txt`, then it's almost impossible to undo your steps beyond restoring an old commit.

You'd have to:
1. Maintain a `pyproject.toml` or other top-level dependency file (`requirements.in`).
2. Recompile to a `requirements.txt` file every time you changed it.
3. Reinstall your virtual environment every time you recompiled.

Fortunately, [Astral's `uv`](https://docs.astral.sh/uv/) automates this process, and adds a lot of extra features and robustness on top!

# The Actual Scariest Command in Python

I said `pip` was the scariest command in Python, but that's not quite true. It's second to the rarer `python3`, the _system Python_!

If you see someone use `python3` for anything other than creating a virtual environment (`python3 -m venv`) then they're working with a common Python environment across their entire computer. If they ever work on two projects, installations from one project will apply to the other! `pip` will ignore any conflicts with existing packages and go nuts on creating an inconsistent environment.

You might return to a project and find that it suddenly doesn't run due to something that happened in an entirely separate Python project. One bad `pip install` could brick any other Python code.
