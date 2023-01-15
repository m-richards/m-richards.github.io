---
title: "Exploring the cross platform dependency management situation in Python"
date: 2023-01-14T21:32:33+11:00
draft: true
---

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
on windows, so is listed with a platform constraint (our codebase is a mini-monolith at this point and so there portions which run quite happily on windows despite missing a "requirement"). `jupyterlab` is cross platform but have some windows specific dependencies which show up if I use `pip-compile`
to generate a requirements file both linux and windows:

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


Realising this limitation of building a simple, clean process around piptools was a bit of a let down. I'd invested in some
tooling precisely to avoid manually fiddling with requirements files. It also seemed to me to be a strange limitation,
why would dependency resolution be dependent on the operating system? It seems like the metadata pip collates from
`pyproject.toml`/ `setup.cfg` / `setup.py` around dependencies of a package should be queryable on any platform, even if its not installable on any package.
That is what prompted this blog post and started me down this rabbit hole.



# Solutions


## Quick and dirty
The simplest solution is to essentially do what I've just done above to illustrate the problem. Run pip compile on WSL against
a windows and linux virtualenv, merge the differences and manually propagate the platform markers. This is manual, error prone and requires either mutliple machines or a specific setup in wsl. Whilst this would be automatable to an extent it's possible to run into situations where for example, considering numpy as a transitive dependency, it ends up pinned to one release on windows, but then say jax imposes an additional constraint on linux and then you've got inconsistency between your dev and deploy environments (if it wasn't clear earlier, this is another downside of maintaining split requirements files per platform as pip tools suggests).

Ideally, a solution would be constructed directly taking into account multiple platforms as part of a single solve. I thought
it was worth taking a quick dive into the internals of pip-tools to work out why this isn't the case

## Pip-tools dependency resolver
<!-- TODO https://github.com/haideralipunjabi/hugo-shortcodes/tree/master/github, github shortcode -->
It's not so tricky to peek under the covers of `pip-tools`. I cloned the repo and first started looking the console 
script entry points for pip-sync and pip-compile. Here they are in the `pyproject.toml`:
```toml
[project.scripts]
pip-compile = "piptools.scripts.compile:cli"
pip-sync = "piptools.scripts.sync:cli"
```
Having a look inside (`scripts.compile.py`)[https://github.com/jazzband/pip-tools/blob/0c1ddcc6466c255675fa8b4f3db7d68f8808a74d/piptools/scripts/compile.py]
 we can see that that compile is built as a (`click`)[https://click.palletsprojects.com/en/8.1.x/] command line tool.

To really understand what's going on, I'd like to replicate my original example and hook into it with a debugger,
so I'm looking to call `pip-compile` programatically. Taking a look through the unit tests, I can see that piptools uses the `CliRunner` class 
defined as a global fixture in `conftest.py`. It seems the simplest way to proceed is to hack into the tests as is. Modifying
`test_cli_compile.py` I have a way in with a debugger:
```python
def test_entry_point(runner):
    with open("requirements.in", "w") as f:
        print('jaxlib;sys_platform != "win32"', file=f)
        print('jupyterlab>=3', file=f)
    runner.invoke(cli, ['requirements.in'])
```

Chopping out the relevant bits of `piptools.scripts.compile::cli`, we have
```python
repository: BaseRepository
repository = PyPIRepository(pip_args, cache_dir=cache_dir)
constraints: list[InstallRequirement] = []
    for src_file in src_files:
        constraints.extend(
                parse_requirements(
                    src_file,
                    finder=repository.finder,
                    session=repository.session,
                    options=repository.options,
                )
            )
```

So `parse_requirements` is where the magic happens. If we look inside again, we see
```python
from pip._internal.req import InstallRequirement
from pip._internal.req import parse_requirements as _parse_requirements
def parse_requirements(
    filename: str,
    session: PipSession,
    finder: PackageFinder | None = None,
    options: optparse.Values | None = None,
    constraint: bool = False,
    isolated: bool = False,
) -> Iterator[InstallRequirement]:
    for parsed_req in _parse_requirements(
        filename, session, finder=finder, options=options, constraint=constraint
    ):
        yield install_req_from_parsed_requirement(parsed_req, isolated=isolated)
```

which offers some clarity as to the underlying question about why this process is platform dependent. Piptools relies on
pip for dependency resolution - and so it must be pip imposing this constraint. Stepping through the pip internals gets 
a bit complicated, so sparing the gory details, in the end a function called `parse_req_from_line` parses a requirements
file string like `jaxlib;sys_platform != "win32"` and splits that into a name, and a `Marker`, and then the name portion
is further parsed into a `Requirement` consisting of a name, `SpecifierSet`(which contains the information on 
versions after the relational operator) and Marker. Eventually this is all bundled up into an `InstallRequirement`
along with a bunch of other metadata - and that forms the list of `constraints` up above.

- invoke the pip resolver directly, discard packages from a requirements file in the same way that would happen for 
installation



# Approaches in other tools





https://pdm.fming.dev/latest/usage/advanced/#export-requirementstxt-or-setuppy
https://frostming.com/2021/03-26/pm-review-2021/