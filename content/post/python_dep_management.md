---
title: "Exploring the cross platform dependency management situation in Python - piptools"
date: 2023-01-14T21:32:33+11:00
draft: true
---

*I've chosen to split this post into at least two parts, as the preamble to give context became a blog post in itself.
So this first piece focuses the context around introducing stricter dependency management, and outlining the cracks that
appear when trying to come up with a solution that works on multiple platforms.*

Recently, I've been looking into transitioning a project (in this case, some transport model dev) from a heavy development
into a production like state. The main goals of such a shift are to reduce inconsistencies between different developers,
their local hardware and the deployed environment where official results are produced. It's also a reflection of the 
of the maturity of the project, a lot has changed over the course of two years of development. 

There are a number of things we're implementing to make this transition, but I thought I'd share a little bit on the 
dependency management situation. There are already a number of blog posts out there comparing `poetry`, `pipenv` and `pip-tools`
so I won't repeat what others have already said better (but if you are interested in how they compare, ideology and 
strong opinions, I'd recommend 
[Should You Use Upper Bound Version Constraints?](https://iscinumpy.dev/post/bound-version-constraints/) and 
[Python Application Dependency Management](https://hynek.me/articles/python-app-deps-2018/) which I find rather compelling). 
I elected to settle on [`piptools`](https://pip-tools.readthedocs.io/en/latest/) as it's relatively lightweight
and just solves the dependency management/ lockfile problem I'm trying to solve and not 15 other things at the same time.

I suppose I should also address another question which comes to mind. Why bother? Why go through the hassle of researching
and setting up a tool when `pip freeze | requirements.txt` probably would have got me 4/5th of the way there? Working at a firm where most people would hesitate to call themselves software developers, this definitely would have been the path of least resistance. But there are a few key reasons that come to mind. To summarise these succinctly;

* It is difficult to track transitive dependencies separately to direct dependencies
  * In turn this makes it difficult to keep transitive deps up to date
  * This also means one is less likely to get security patches in a timely fashion
* Determining if a dependency is no longer required becomes hard
* The status quo becomes avoiding updating dependencies if at all possible (which is short sighted, but remains a tempting business decision in the world of consulting. )

Collectively, these amount to the situation where either packages are almost never updated, or every request to update a package or refresh dependencies comes through me. Which is not a workflow I'm especially keen on supporting for the rest of time. Fortunately, through a combination of `pip-tools`
some wrapper tooling, and a healthy amount of documentation I've managed to cobble together a solution which I'm optimistic
will provide a simple way for the project team to interact with requirements files and avoid situations like working with
`geopandas 0.5` in 2021.


# Cross platform dependencies - the problem
With preamble aside, I wanted to share an aspect of the process that I'm still not quite happy with, and dig into it a 
little bit, as on the surface it seems rather perplexing that this isn't a solved problem.

To explain in more detail, let's consider the following unpinned dependencies, which correspond to a `requirements.in` file
in the piptools model. 
<!-- TODO, find a way to render code block titles -->
```python {title="requirements.in"}
jaxlib;sys_platform != "win32"
jupyterlab>=3
```

The packages in question are a little confected, but they illustrate the situation quite nicely. Jax is not supported
on windows, so is listed with a platform constraint (our codebase is a mini-monolith at this point and so there portions which run quite happily on windows despite missing a "requirement"). `jupyterlab` is cross platform but has some windows specific dependencies which show up if I use `pip-compile`
to generate a requirements file. Here's want that looks like if I compare the pip-compile output for linux and windows:

```bash
$ pip-compile --no-annotate --output-file=requirements_windows.txt --resolver=backtracking requirements.in
$ pip-compile --no-annotate --output-file=requirements_linux.txt --resolver=backtracking requirements.in
$ git diff requirements_windows.txt requirements_linux.txt -U0
```
```diff
# (cleaned up a bit manually)
-colorama==0.4.6
-pywin32==305
-pywinpty==2.0.10
+jaxlib==0.4.1 ; sys_platform != "win32"
+numpy==1.24.1
+pexpect==4.8.0
+ptyprocess==0.7.0
+scipy==1.10.0
```

Firstly, its clear that results are platform dependent. This is clearly noted in the pip-tools documentation, along with
the suggestion that one should have a requirements file per platform, python version, cpu type etc. This however isn't
a solution that scales well, especially if one already has more than one target requirements file, which is not so
unusual in the context of a repository. Secondly, we notice that the platform markers don't propagate. `jaxlib` pulled in 
`numpy`, `pexpect`, `ptyprocess` and `scipy`, without the restriction that these were transitive dependencies of a linux
only package. Fortunately, these all install on windows (even if they're not intended for or tested on the platform) but 
the situation for windows dependencies is not so kind. `pywin32` and `pywinpty`, these packages don't make sense nor exist
for linux, so the requirements file is not installable on linux.


Realising this limitation of piptools was a bit of a let-down. I now had some nice clear easy to use tooling
with an ugly manual hack to handle at the end. It also seemed to me to be a strange limitation,
why would dependency resolution be dependent on the operating system? It seems like the metadata pip collates from
`pyproject.toml`/ `setup.cfg` / `setup.py` around dependencies of a package should be queryable on any platform, even if its not installable on any package.
That question is what prompted this blog post to see if there's a better answer than "it's not supported, cope".

The second part of this will dive deeper into the rabbit hole of why piptools works how it does, and whether my naive 
expectation that this problem should be solvable stacks up. I also have a look at how some of the contemporaries to piptools behave
in the same situation.
