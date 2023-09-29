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

When your project have a lot of dependedises it may quite often happen that one of them will release a new version that will break your tests. Without pining test dependecies it will imeadetly break your CI.
It is especially problematic if you have multple contributtors outside of your organization. They may not be aware of the problem and will not be able to fix it.

### Reproducibility in science

Stroing pinned test dependecise int the repository means also storing pinned runtime dependecies. This allow someone install project in older version that may be required to reproduce research results.