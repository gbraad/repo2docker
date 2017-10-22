# jupyter-repo2docker


[![Build Status](https://travis-ci.org/jupyter/repo2docker.svg?branch=master)](https://travis-ci.org/jupyter/repo2docker)

**jupyter-repo2docker** is a tool to build, run and push docker images from source code repositories.


## Pre-requisites

1. Docker to build & run the repositories. The [community edition](https://store.docker.com/search?type=edition&offering=community)
   is recommended.
2. Python 3.4+.

## Installation

To install from pypi, the python packaging index:

```bash
pip install jupyter-repo2docker
```

To install from source:

```bash
git clone https://github.com/jupyter/repo2docker.git
cd repo2docker
pip install .
```

## Usage

The core feature of repo2docker is to fetch a repo (from github or locally), build a container
image based on the specifications found in the repo & optionally launch a local Jupyter Notebook
you can use to explore it.

**Note that Docker needs to be running on your machine for this to work.**

Example:

```bash
jupyter-repo2docker https://github.com/jakevdp/PythonDataScienceHandbook
```

After building (it might take a while!), it should output in your terminal something like:


```
    Copy/paste this URL into your browser when you connect for the first time,
    to login with a token:
        http://0.0.0.0:36511/?token=f94f8fabb92e22f5bfab116c382b4707fc2cade56ad1ace0
```

If you copy paste that URL into your browser you will see a Jupyter Notebook with the
contents of the repository you had just built!

### Displaying the image Dockerfile
Repo2Docker will generate a Dockerfile that composes the created Docker image.
To see the contents of this Dockerfile without building the image use `--debug` and `--no-build`
flags like so:

```bash
jupyter-repo2docker --debug --no-build https://github.com/jakevdp/PythonDataScienceHandbook
```

## Repository specifications

Repo2Docker looks for various files in the repository being built to figure out how to build it.
It is philosophically similar to [Heroku Build Packs](https://devcenter.heroku.com/articles/buildpacks).

It currently looks for the following files. They are all composable - you can use any number of them
in the same repository (except for Dockerfiles, which take precedence over everything else).

### `requirements.txt`

This specifies a list of python packages that would be installed in a virtualenv (or conda environment).

### `environment.yml`

This is a conda environment specification, that lets you install packages with conda. Note that you must
leave the name of the environment empty for this to work out of the box.

### `apt.txt`

A list of debian packages that should be installed. The base image used is usually the latest released
version of Ubuntu (currently Zesty.)

### `postBuild`

A script that can contain arbitrary commands to be run after the whole repository has been built. If you
want this to be a shell script, make sure the first line is `#!/bin/bash`. This file must have the
executable bit set (`chmod +x postBuild`) to be considered.

### `REQUIRE`

This specifies a list of Julia packages! Currently only version 0.6 of Julia is supported, but more will
be as they are released.

### `Dockerfile`

This will be treated as a regular Dockerfile and a regular Docker build will be performed. The presence
of a Dockerfile will cause all other building behavior to not be triggered.

## Design

`repo2docker` has two primary use cases, which drive most design decisions.

1. Automated image building with projects like
   [BinderHub](http://github.com/jupyterhub/binderhub)
2. Manual image building + running using the `jupyter-repo2docker` commandline
   client on user's interactive workstations.

We enumerate some of these design principles here. This is not an exhaustive
list :)

### Deterministic output

The core of `repo2docker` can be considered a
[deterministic algorithm](https://en.wikipedia.org/wiki/Deterministic_algorithm).
It takes as input a directory which has a repository checked out, and
deterministically produces a Dockerfile based on the contents of the directory.
So if we run repo2docker on the the same directory multiple times, we get the
exact same Dockerfile output.

This provides a few advantages:

1. We can cache the built artifacts based on the identity of the repository we are
   building. For example, if we had already run repo2docker on a git repository
   at a particular commit hash, we know we can just re-use the old output, since
   we know it is going to be the same. This provides massive performance &
   architectural advantages when building additional tools (like BinderHub) on
   top of repo2docker.
2. We produce Dockerfiles that have as much in common as possible across
   multiple repos, enabling better use of the Docker build cache. This also
   provides massive performance advantages.

### Unix principles

`repo2docker` should do one thing, and do it well. This one thing is:

> Given a repository (of some sort), deterministically build a docker image from
> it.

There's also some convenience code (to run the built image) for users, but
that's separated out cleanly. This allows easy use by other projects (like
BinderHub).

There is additional (and very useful) design advice on this in
the [Art of Unix Programming](http://www.faqs.org/docs/artu/ch01s06.html) which
is a highly recommended quick read.

### Composability

The prime reason `repo2docker` exists (rather than just using something
like [s2i](https://github.com/openshift/source-to-image)) is we want to support
*composable* environments. We want to easily have an image with
Python3+Julia+R-3.2 environments, rather than just one single language
environment. While generally one language environment per container works well,
in many scientific / datascience computing environments you need multiple
languages working together to get anything done. So all buildpacks are
composable, and need to be able to work well with other languages.

### [Pareto principle](https://en.wikipedia.org/wiki/Pareto_principle)

Roughly speaking, we want to support 80% of use cases, and provide an escape
hatch (raw Dockerfiles) for the other 20%. We explicitly want to provide support
only for the most common use cases - covering every possible use case never ends
well.

An easy process for getting support for more languages here is to demonstrate
their value with Dockerfiles that other people can use, and then show that this
pattern is popular enough to be included inside repo2docker. Remember that 'yes'
is forever (very hard to remove features!), but 'no' is only temporary!
