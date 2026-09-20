---
tags: [dashboard, python, install, pip]
---

# Installing `ops_stats`

> [!info] Purpose
> Quick setup so any application that needs to report state/stats can
> install the `ops_stats` package correctly.

## Check you're using the right `pip`

On macOS it's easy to accidentally install a package for the wrong
Python (a Homebrew Python, the system Python, a different pyenv
version). Always invoke `pip` **through the Python interpreter you'll
actually run the app with**, rather than calling `pip` directly:

```bash
python3 -m pip --version
```

This prints both the pip version and the Python it's tied to — check
that path matches the interpreter your application runs on. If you use
a virtual environment (recommended), activate it first:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
```

> [!tip]
> `python3 -m pip install ...` is safer than plain `pip install ...` —
> it guarantees the package lands in the environment tied to that
> specific `python3`, not whichever `pip` happens to be first on your
> `PATH`.

## Add it to a requirements file

Keep dependencies declared in a `requirements.txt` rather than
installing ad hoc, so any machine (or teammate) can reproduce the same
environment:

```
ops_stats>=1.0
```

> [!warning]
> Replace `>=1.0` with whatever version scheme actually applies once
> `ops_stats` has a real release — and confirm where it's installed
> *from* (PyPI, an internal package index, or a git URL) if it isn't
> published publicly yet. If it's a private git repo, the line looks
> like:
> ```
> https://github.com/platform-42/p42_dashboard.git
> ```

## Install from the requirements file

```bash
python3 -m pip install -r requirements.txt
```

Re-run the same command any time `requirements.txt` changes — pip only
installs/upgrades what's needed.

## See also

- [[Using ops_stats from Python]]
