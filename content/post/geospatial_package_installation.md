---
title: Tips for success installing python packages
date: 2023-05-27T12:09:28+10:00
draft: false
categories:
  - Python
  - Geospatial
  - Teaching
---

Although python has a functional story around packaging, there are plenty of footguns for people who only occasionally
dabble with package management. This situation is fairly common in my line of work, when a project is started, the
project team get set up with working developer environments, and then happily forget about packaging problems for the
next x months until the next project starts. So this is a quick reference I hope to point people to the next time I get
*"my python environment isn't working / I can't install package x questions"*. Given I usually fail on the "quick
reference" front, I'm going to try be light on detail and just present sensible default choices.

**Aside on geospatial packages**: Part of the complication with package installations is that many python packages depend
on underlying C or C++ libraries. Historically this meant that installing `gdal` or anything depending on it (such as
fiona, geopandas, pyogrio) required a C compiler installed on windows, and that led to impression of conda being the "easy way" to get these packages. As of 2022 however, `pyogrio` and `fiona` have binary wheels for windows, meaning the
low level dependencies are bundled in and `pip install fiona` just works. (I could probably say a fair bit more on the details of this, but I'll save it for another time)

# Using conda/ mamba

I wouldn't say using conda / mamba is necessarily better/ easier than pip, it's just fresh in mind because I recently
helped a colleague using conda. Perhaps I'll revisit at some point with a pip analogue. First I have a list of
recommendations, and then an example of how to enact them:

## Recommendations

1. **Don't use the `base` environment! / Always create a dedicated environment for a project.** Environments are cheap, wasting time
   debugging weird inconsistencies from something you installed years ago is not
2. **Use the conda-forge channel.** Conda can download from multiple places, by default it uses the `defaults` channel,
   which often has packages out of date by 6-12 months. It also generally has less packages available than
   `conda-forge`, so its fairly safe to recommend `conda-forge` by default (there's also legal/ licensing shenanigans about using the `defaults` channel at large organisations). You can ensure conda forge is used by default globally by running
   `conda config --prepend channels conda-forge` and `conda config --set channel_priority strict`.
3. **When your environment is working, serialise it.** Don't end up in the situation where you've finally got everything
   working, you install something else and everything spontaneously combusts. (This hasn't happened to me in a long
   time, but it really sucks when it does, and avoiding it is pretty easy). Run `conda activate <envname>` and
   `conda env export > "<env_name.yml>"` to create a YAML encoding dependencies (in the current working directory). Then commit this file! Don't make someone else have to rediscover what a working environment looks like. the `--from-history` flag can also be used in `conda
   env export` to ignore transitive dependencies, which might be handy in particular if dependencies are different on
   different platforms.
4. **Don't mix pip and conda unless you have to**. If you have to (a package isn't available on conda-forge) the
   recommendation is to only use pip from that point forward. The two ways of installing packages work in different ways
   with different metadata, and installing conda packages after this point can brick an environment. (If you are naughty
   and ignore this recommendation, the prospect of a bricked environment is much less scary if you have a yaml to
   faithfully re-create to the state before it was ruined. Also, bricking seems to happen less frequently than it used
   to, at least for me, although I don't use conda as much these days.)
5. **Use the existing environment specification if there is one.**
   - If a pip style `*.txt` exists, use `conda create -n <ENVNAME> --file <ENV_FILENAME>.txt`
   - If a conda YAML exists use `conda env create -n <ENVNAME> --file <ENV_FILENAME>.yml`

## A practical example

I've recently been through this process for a repository where no one left a helpful `requirements.txt` to get us started with. Instead, we have to re-create the environment
based upon imports in the code or other detective work. I can see that gdal is required, so I'll include that in creating the environment, along with a
couple of other packages that I'll likely need as well. Ideally, I'd know exactly what packages are required so conda
can resolve everything in one step, but that's not always an option.

```bash
conda create -n <my_env_name> -c conda-forge python=3.10 gdal geopandas matplotlib jupyter lab
```

(Note that if you followed the tip on setting conda-forge as the default channel, you can omit the `-c conda-forge`
part.) Then activate the environment to attempt to run the notebook in the repository:

```bash
conda activate <my_env_name>
jupyter lab
```
Activating the environment is important - we want jupyter lab to use the environment with dependencies. (There's also [nb-conda-kernels](https://github.com/Anaconda-Platform/nb_conda_kernels) that's supposed to let you keep jupyterlab in one place only, but I've been burnt by this not working and not loading DLLs in the past)

Later we discovered we needed a pip package `pygdaltools`, so we installed that as well,

```bash
pip install pygdaltools
```

Finally, once everything was working, we can export the environment with

```bash
conda env export > "<env_name.yml>"
```

and commit to the repository, to spare anyone else this exercise in future.
