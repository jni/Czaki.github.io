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

# Pinning Test Dependencies

## Motivation

### Guaranteeing test stability

As one of the principal developers of the [napari](https://github.com/napari/napari) project, it is noteworthy to mention that at the time of writing, this project encompasses 36 direct dependencies and 116 total dependencies for minimum operational capability. In terms of testing, it utilizes 165 dependencies in total, while the assembly of documents requires up to 191 dependencies.

Given the substantial volume of these dependencies, there are instances where the maintainers of other libraries might unintentionally disrupt certain API in their packages during updates. These occasional or unintentional disruptions can cause unforeseen complications. On the other hand, there are times when such breakages are deliberate. Providing a backward compatibility layer can be tricky in these situations. Balancing the maintenance of deprecated APIs with both reasonable time constraints and performance requirements often proves unfeasible in the long run. These factors illuminated the need for us to commence pinning our test dependencies.

Prior to implementing the pinning of test dependencies, we frequently encountered issues with broken tests, roughly about once a month. Many of our contributors are novices in programming and contributing to open-source projects. They often struggle with understanding why tests fail and whether it's an outcome of their modifications. Thus, alongside fixing the test, we have to embark on additional maintenance work to clearly explain the circumstances leading to the failure.

### Enhancing reproducibility in science-based projects

Napari, being in its alpha phase, is substantially a science-based project. As such, we occasionally have to make API modifications. Consequently, newer library updates may not be compatible with older code. However, in scientific research, there's often a need to re-run older codes for comparison purposes.

Luckily, there are project tools like [`pypi-timemachine`](https://pypi.org/project/pypi-timemachine/) that design proxy servers to retrieve PyPi packages up till a specified date. Despite this, providing a definitive list of constraints enables the environment to be easily replicated without resorting to additional tools and avoiding discrepancies that could impact results. Also, this list of constraints simplifies the process of instructing others on how to recreate the environment.

## Distinguishing between pip's Requirements and Constraints

In pip, requirements are a list of packages that need to be installed and could contain version constraints and [extras](https://pip.pypa.io/en/stable/reference/requirement-specifiers/#requirement-specifiers) definitions. On the other hand, constraints are version boundaries for packages that should be considered during the resolution process. 

Requirements can be passed by using the package name like `pip install numpy`, or via a file like `pip install -r requirements.txt`. Constraints, however, can be passed using the file `pip install napari -c constraints.txt` or via environment variables `PIP_CONSTRAINT=constraints.txt pip install napari`. It's important to note that putting packages in constraints.txt does not install them. It only limits the versions that could be installed.

## What we need
 
Here is a list of our needs:
 
1. We need to pin all dependencies for tests and docs.
2. We need to pin per Python version, as some dependencies do not provide single-version support for all Python versions that we support.
3. Constraints need to be automatically updated (creating Pull Requests).
4. As we use upper constraints for some dependencies, we need to be able to upgrade dependencies in any PR, that comes from a forked repository. 
5. We need to minimize the infrastructure cost for this process.
 
## Our solution
 
Here, I will describe our solution to the problem. Today's date is 2023.11.7. Workflow may change over time. The most recent version of the workflow is available [here](https://github.com/napari/napari/blob/main/.github/workflows/upgrade_test_constraints.yml).
