---
title: Setting up a pandas/main environment for GeoPandas
date: 2023-02-05T18:05:38+11:00
draft: false
categories:
  - Python
  - Geospatial
  - Teaching
---

Although it's not something I tend to advertise very often, I'm a maintainer for
[GeoPandas](https://github.com/geopandas/geopandas) which is the de-facto standard tool for tabular geospatial analysis
in Python. My contributions here wax and wane here with the amount of free time and mental space I have to volunteer. As
part of this, we keep the software in sync with developments in [pandas](https://github.com/pandas-dev/pandas), and our
CI tests against `pandas/main`. From time to time (more frequently with the upcoming pandas 2.0 release) it's necessary
to test locally again the main branch, make use of a debugger, and work out how best to migrate the code towards an
upcoming change. This short post is just some reminders for me for setting up this process. I'm borrowing heavily the
respective [pandas](https://pandas.pydata.org/docs/development/contributing_environment.html) and
[GeoPandas](https://geopandas.org/en/latest/community/contributing.html#creating-a-development-environment) contributor
guides, but slicing down to the bits I care about (and changing the bits I don't like).

# Environment Setup

I've been using [mambaforge](https://github.com/conda-forge/miniforge#mambaforge) for environment management outside of
work for quite a while, and these instructions are assuming that lightly (note, from normal conda, the biggest implicit
assumption is that the default channel is conda-forge).

1. Clone pandas `git clone git@github.com:<fork_author>/pandas` and
   `git remote add upstream git@github.com:pandas-dev/pandas`
2. `cd` into pandas fork checkout. `git fetch upstream` and `git merge upstream/main`
3. `git fetch --all --tags`. This is sometimes a gotcha when dealing with compatibility code based on pandas version. In
   a dev environment `pandas.__version__` is git aware.
4. `mamba env create -n {dev_env_name} python=3.11 Cython versioneer pytest pytest-xdist numpy python-dateutil pytz matplotlib pyarrow scipy numpy`,
   I use this manual list as the `environment.yml` has a zoo of optional dependencies I don't overly care about from a
   GeoPandas context, and I'd rather use a python more recent than 3.8.
5. `mamba activate pandas-dev`
6. `python setup.py build_ext -j 4` (build the cython extension modules, with 4 cores)
7. `python -m pip install -e . --no-build-isolation --no-use-pep517`
8. `pre-commit install` (technically optional if only working on geopandas)

## Add GeoPandas on top

1. `mamba install -y fiona pyproj shapely pyogrio black pre-commit ipython jupyterlab`
2. cd to geopandas fork dir and `python -m pip install -e .`
3. `pre-commit install`

At this point, if I've done everything right, I should be able to run the geopandas test suite mostly successfully.

# Reminder: how to update the pandas code

Despite being reasonably happy reading and editing (and on rare occasion, writing) cython, I never seem to dig into it
enough to remember what the right invocation for the compile step is. This is my self reminder:

1. `cd` into pandas fork checkout. `git fetch upstream --tags` and `git merge upstream/main`
2. `git checkout main` and `git pull`
3. `python setup.py build_ext -j 4` - this should indicate it's cython-ising a bunch of things Note that this last on
   windows will often produce a bunch of warnings and sometime fail with exit code 1, even if it succeeds.
4. `python -m pip install -e . --no-build-isolation --no-use-pep517`

(Sometimes, I've also had this stuck in a bad state and running `python setup.py develop` has gotten things back. This
should be mostly equivalent to the pip install editable though)

# Extra: pyogrio dev env on windows with OSGeo4W.

*After experimenting with this, I've reverted to using WSL as it seems less flaky overall, ended up with a broken
environment down the track and not exactly sure why* These are my notes on installing pyogrio from source on windows,
which flesh out the notes in (the docs)\[https://pyogrio.readthedocs.io/en/latest/install.html#windows\]. I've done this
most recently with GDAL 3.6.4 from OSGeo4W with QGIS 3.30, but also with GDAL 3.5.1 in the past.

01. Download OSGeo4W network installer https://www.qgis.org/en/site/forusers/download.html
02. (As administrator) run installer for all users, install gdal and gdal-devel (the latter adds header files and
    populates the \\include dir)
03. Create conda env
    `conda create -n pyogrio_dev python=3.11 pandas shapely Cython pyproj ipython pytest pyarrow versioneer`. (**Do not
    install fiona!** - this will cause DLL loading errors from the conflicting versions of GDAL. Perhaps this can work
    if building fiona from source as well, but i haven't tried.)
04. Activate the environment: `conda activate pyogrio_dev`
05. In OSGeo4W shell, run `gdalinfo --version` we need to know the version of GDAL to pass to the installler.
06. Switch to dir containing checkout of pyogrio
07. Install pyogrio
    `python -m pip install --install-option=build_ext --install-option="-IC:\OSGeo4W\include" --install-option="-lgdal_i" --install-option="-LC:\OSGeo4W\lib" --no-deps --no-use-pep517 --install-option=--gdalversion --install-option=3.6.4 -e . -v`
    (where you replace 3.6.4 with whatever version of gdal is reported by gdalinfo). Note this looks a bit odd supplying
    `--gdalversion` and `3.6.4` separately, but the pyogrio setup code looks specifically for the key `--gdalversion`,
    so we have to pass these as two consecutive arguments.
08. Alternatively, set environment variables
    `$env:GDAL_VERSION="3.6.4"; $env:GDAL_LIBRARY_PATH="C:\OSGeo4W\lib"; $env:GDAL_INCLUDE_PATH="C:\OSGeo4W\include"`
    and run `python -m pip install --no-deps --force-reinstall --no-use-pep517 -e . -v`
09. You might have to set the environment variable `GDAL_DATA`. I've now set this to
    `$env:GDAL_DATA="C:\OSGeo4W\apps\gdal\share\gdal"`, but I remember this "just working" in the past.
10. If everything has gone well, importing pyogrio will work and the tests will pass when run.

## pip 23.1 compatibility

In pip 23.1, the `--install-option` flag in pip was removed. For now, it seems that using `--config-settings` (the
apparent replace) doesn't behave. Instead, supply the environment variables as in (9). There's potentially some work to
do on the packaging of pyogrio to make this a little easier, but not a packaging expert.

# Extra: pyogrio linux install

1. conda env
   `mamba create -n pyogrio_dev python=3.11 pandas shapely Cython pyproj ipython pytest pyarrow versioneer gdal`
2. clone pyogrio & fetch tags
3. Get us a GDAL to build against:
   - Using apt: `sudo apt install gdal-bin` and `sudo apt-get install libgdal-dev`
   - Using conda: *I'm yet to actually test this directly because it seems to require there to not be a system GDAL /
     system GDAL not on path, and I can't do that without breaking my existing environment*. Originally I presumed this
     wouldn't install the requisite header files to build other packages against, but pyogrio does this in its CI. In
     theory though, this is great because you're not tied to the version of GDAL bundled into debian stable, and don't
     have to updated ubuntu to get a new version of GDAL. `mamba install gdal`
4. `python setup.py develop`
5. `pip install --no-deps geopandas` - don't want to install fiona which has another version of GDAL bundled into the
   wheel (could perhaps use conda/ mamba to install this too)
