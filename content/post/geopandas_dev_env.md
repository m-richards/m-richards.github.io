---
title: "Setting up a pandas/main environment for GeoPandas"
date: 2023-02-05T18:05:38+11:00
draft: false
categories:
  - Python
  - Geospatial
  - Teaching
---

Although it's not something I tend to advertise very often, I'm a maintainer for [GeoPandas](https://github.com/geopandas/geopandas)
which is the de-facto standard tool for tabular geospatial analysis in Python. My contributions here wax and wane here
with the amount of free time and mental space I have to volunteer. As part of this, we keep the software in sync
with developments in [pandas](https://github.com/pandas-dev/pandas), and our CI tests against `pandas/main`. From
time to time (more frequently with the upcoming pandas 2.0 release) it's necessary to test locally again the main branch,
make use of a debugger, and work out how best to migrate the code towards an upcoming change. This short post is 
just some reminders for me for setting up this process. I'm borrowing heavily the respective [pandas](https://pandas.pydata.org/docs/development/contributing_environment.html) and [GeoPandas](https://geopandas.org/en/latest/community/contributing.html#creating-a-development-environment) contributor guides,
but slicing down to the bits I care about (and changing the bits I don't like).

# Environment Setup

I've been using [mambaforge](https://github.com/conda-forge/miniforge#mambaforge) for environment management
outside of work for quite a while, and these instructions are assuming that lightly (note, from normal conda, the 
biggest implicit assumption is that the default channel is conda-forge). 

1. `cd` into pandas fork checkout. `git fetch upstream` and `git merge upstream/main`
2. `git fetch --all --tags`. This is sometimes a gotcha when dealing with compatibility code based on pandas version. In a dev environment `pandas.__version__` is git aware.

3. `mamba env create -n python=3.11 Cython versioneer pytest pytest-xdist numpy python-dateutil pytz matplotlib pyarrow scipy`, I use this manual list as the `environment.yml` has a zoo of optional dependencies I don't overly care about from a GeoPandas context, and I'd rather use a python more recent than 3.8. 
4. `mamba activate pandas-dev`
5. `python setup.py build_ext -j 4` (build the cython extension modules, with 4 cores)
6. `python -m pip install -e . --no-build-isolation --no-use-pep517`

## Add GeoPandas on top
1. `mamba install -y fiona pyproj shapely pyogrio black pre-commit ipython jupyterlab`
2. cd to geopandas fork dir and `python -m pip install -e .`

At this point, if I've done everything right, I should be able to run the geopandas test suite mostly successfully.

# Reminder: how to update the pandas code
Despite being reasonably happy reading and editing (and on rare occasion, writing) cython, I never seem to dig into it 
enough to remember what the right invocation for the compile step is. This is my self reminder:
1. `cd` into pandas fork checkout. `git fetch upstream --tags` and `git merge upstream/main`
2. `git checkout main` and `git pull`
3. `python setup.py build_ext -j 4` - this should indicate it's cython-ising a bunch of things

Note that this last on windows will often produce a bunch of warnings and sometime fail with exit code 1, even if it succeeds.

(Sometimes, I've also had this stuck in a bad state and running `python setup.py develop` has gotten things back.)






