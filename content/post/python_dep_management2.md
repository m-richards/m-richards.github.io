---
title: 'Exploring the cross platform dependency management situation in Python - Part
  2: piptools internals'
date: 2023-01-15T21:45:40+11:00
draft: false
---

This is a continuation of \[my previous post about `pip-tools`\]({{\< ref "python_dep_management.md" >}}), and the
unsatisfying conclusion that it doesn't handle generating cross platform requirements particularly elegantly. I started
this investigation with the premise that it should be possible to generate a cross-platform environment specification
from a single computer. This post determines the validity of that premise, by taking a peek at the internals of the
`pip-tools` and its dependency solver.

## Pip-tools dependency resolver - under the covers

Refresher: I was using `pip-compile` to generated locked `requirements.txt` files from unpinned `requirements.in` files.
I'm now looking to understand the mechanism for how that works.

<!-- TODO https://github.com/haideralipunjabi/hugo-shortcodes/tree/master/github, github shortcode -->

It's not so tricky to peek under the covers of `pip-tools`. I cloned the repo and first started looking the console
script entry points for pip-sync and pip-compile. Here they are in the `pyproject.toml`:

```toml
[project.scripts]
pip-compile = "piptools.scripts.compile:cli"
pip-sync = "piptools.scripts.sync:cli"
```

Having a look inside
[`scripts.compile.py`](https://github.com/jazzband/pip-tools/blob/0c1ddcc6466c255675fa8b4f3db7d68f8808a74d/piptools/scripts/compile.py)
we can see that that compile is built as a [`click`](https://click.palletsprojects.com/en/8.1.x/) command line tool.

To really understand what's going on, I'd like to replicate my original example and hook into it with a debugger, so I'm
looking to call `pip-compile` programatically. Taking a look through the unit tests, I can see that piptools uses the
`CliRunner` class defined as a global fixture in `conftest.py`. It seems the simplest way to proceed is to hack into the
tests as is. Modifying `test_cli_compile.py` I have a way in with a debugger:

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
is further parsed into a `Requirement` consisting of a name, `SpecifierSet`(which contains the information on versions
after the relational operator). Eventually this is all bundled up into an `InstallRequirement` along with a bunch of
other metadata - and that forms the list of `constraints` up above.

The interesting thing here is that on the surface, all the information is passed down to the dependency resolver such
that it should be possible to construct a cross platform solution. Originally, I'd planned to walk my way through the
internals and work out where this gets dropped from first principles but the mechanics of doing this and getting useful
information out is more complex than I was hoping\[^1\]. Fortunately, we can benefit from the wisdom from others,
provided we know where to look - in this case to
[Why PyPI Doesn't Know Your Projects Dependencies](https://dustingram.com/articles/2018/03/05/why-pypi-doesnt-know-dependencies/).
Essentially, this is due to historical reasons, where python dependency metadata was specified in `setup.py` which is
executable, and there for can change at runtime. So for pip to know the dependencies associated with a package it has to
install it - and to install it, it has to be installable on your platform - hence the idea of simulultaneously
satisfying windows and linux constraints as part of a single solve doesn't make sense - at least in the pip
universe\[^2\].

Going into this, I did not have any appreciation for the complexities involved in this process, but perhaps I could have
taken a hint from Hatch, PDM, Poetry and pipenv all competing in a very similar space around python packaging. I had
originally planned to take a dive into other some of these other tools as well (I know for a fact that PDM will generate
cross platform lockfiles, and I've heard poetry can do this too) but I plan to park this investigation for now, and
maybe find something less full on to write about in the mean time. If I do make any updates to the lockfile process I've
mentioned, I will share that though.

\[^1\]: `click` redirects stdout and stderr in a convoluted way right at the start of when it invokes a command line
util function, which means you can't see any debug logging in real time. If the situation were simple, that wouldn't
matter too much as ide tools around debugging are pretty good - but it's not, so logging is really helpful. To get
around this I'll need to tear apart the `pip-tools` internals even further and create a non click version of the command
line invocation. I've actually done this now, but my enthusiasm to persevere is rather dampened at the moment.

\[^2\]: `setup.py` is outdated, replaced by `pyproject.toml` (and at some point prior `setup.cfg`) and so is a link to
an article from 2018. So why are things still like this? My understanding on this is not perfect (I've read a lot of
PEPs, github threads, blog posts and still don't have full clarity) but it seems that wheels contain the
["Core metadata specifications"](https://packaging.python.org/en/latest/specifications/core-metadata/#core-metadata),
which can optionally include information about required python version, dependencies and platform. So it's possible to
just download the wheel, extract it and look at the metadata file, and potentially avoid having to install the wheel as
well (in future/ perhaps already this will be available directly on pypi as a separate file, so the wheel unpacking
doesn't need to happen - see [PEP 658](https://peps.python.org/pep-0658/#dist-info)). The way that pip queries this
metadata however is still in the context of trying to install packages on a specific platform - so there's no easy
solution here.
