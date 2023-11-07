---
date: 2023-09-29
categories:
  - GitHub
  - Testing
  - Article
tags:
    - GitHub
    - Testing
    - Article
draft: true
---

# Pinning test dependencies

## Motivation

## Test stability

I'm one of the core-developers of [napari](https://github.com/napari/napari) project. 
At the moment of writing, this project has 36 direct dependencies and 116 total dependencies for minimum working dependencies.
The test uses 165 dependencies in total. Building docs requires 191 dependencies.

This is a lot of dependencies, and their maintainers do not always notice that they will break some use cases. 
Sometimes, breaking change is required, and providing deprecated API for a long time is impossible.

Before we started pinning test dependencies on average once a month, we had problems with broken tests.
Many of our contributors are learning programming and open-source contributions.
So, it is hard for them to understand why tests are failing and whether it is the fault of their changes.
So next to fixing the test, there is also an explanation to perform additional maintenance work to explain to people what happens.


### Reproducibility in science

The napari is a scientific project in the alpha phase. So, we regularly break API. 
An old code may not work with the new library version. 
However, scientific work sometimes requires running old code to compare results.
There are projects like [`pypi-timemachine`](https://pypi.org/project/pypi-timemachine/) 
that create a proxy server that returns PyPi packages until a specific date.

However, providing an explicit list of constraints allows the environment 
to be reproduced without additional tools, finding differences that may lead to different results.
Also, constraints make it much easier to explain to people how to reproduce the environment.


## Difference between requirements and constraints for pip

The requirements are a list of packages that need to be installed. 
It could contain version constraints and [extras](https://pip.pypa.io/en/stable/reference/requirement-specifiers/#requirement-specifiers) definition. 
The constraints are a list of version bounds for packages that should be handled during the resolution process. 

Requirements could be passed using the package name `pip install numpy` or using a file `pip install -r requirements.txt`.
Constraints could be passed using the file `pip install napari -c constraints.txt` or using environment variables `PIP_CONSTRAINT=constraints.txt pip install napari`. 
It will not be installed if a package is mentioned in the constraints but not in the requirements.

## What we need

Here is a list of our needs:

1. We need to pin all dependencies for tests and docs.
2. We need to pin per Python version, as some dependencies do not provide single-version support for all Python versions that we support.
3. Constraints need to be automatically updated (creating Pull Requests).
4. As we use upper constraints for some dependencies, we need to be able to upgrade dependencies in any PR, that comes from a forked repository. 
5. We need to minimize the infrastructure cost for this process.

## Our solution

Here, I will describe our solution to the problem. Today's date is 2023.11.7. Workflow may change over time. The most recent version of the workflow is available [here](https://github.com/napari/napari/blob/main/.github/workflows/upgrade_test_constraints.yml).



