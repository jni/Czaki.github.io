---
date: 2024-09-14
categories:
  - Python
  - Qt
  - Testing 
  - Ci
tags:
    - Python
    - Testing
    - Article
draft: true
---

# Preventing segfaults in test suite that has Qt Tests

# Motivation

When providing an GUI application one needs to select GUI backend. 
If application is Python and needs to work on all popular OSes[^1]
the good choice is to use Qt. It is a cross-platform GUI toolkit that
has good python bindings[^2].

However, the Qt objects require special care when it comes to testing. 
It this post I will describe my experience of writing such tests based on 
my experience from development of [PartSeg](https://partseg.github.io/) 
and [napari](https://napari.org/).




[^1]: This includes Windows, macOS, and various distributions of Linux.
[^2]: PyQt5, PySide2 for Qt5, PyQT6, PySide6 for Qt6.