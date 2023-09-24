---
title: "Tidbits from the Scientific python development guide"
date: 2023-09-10T20:17:43+10:00
draft: false
categories:
  - Python
  - Packaging
---

Recently, the (Scientific Python development guide)[https://learn.scientific-python.org/development/] was (announced)[https://blog.scientific-python.org/scientific-python/dev-summit-1-development-guide/] and released, serving as a consolidation of existing developer guides outlining software development
best practices in the scientific python ecosystem. It's grown out of a collection of materials for (Scikit HEP)[https://scikit-hep.org/]
which I'd first encountered years ago, looking for python support for ragged arrays and noted the very useful documentation pages.

It's very nice to see a consensus backed series of guidelines and first principles in a consolidated location,
as python packaging is esoteric at times, and has a long legacy of deprecated ways of doing things (which often are 
unfortunately still in regular use). It's also great to see the inclusion of tooling to help people towards implementing
the guidelines, and identifying where existing packages depart from those - in particular the (Repo Review tool)[https://learn.scientific-python.org/development/guides/repo-review/], which is a super cool use of WASM. Will definitely be trying this out with geopandas related repositories.

Rather than do any kind of summary of the guide, I thought I'd just make some notes on sections I found interesting:

- Using `__file__` to refer to data files commited alongside code is not robust - that's not surprising. But in particular this can go wrong if
   - The files are missing from the sdist or wheel when you upload your package
   - People are running your package from inside a (zipfile)[https://docs.python.org/3/library/zipimport.html#examples] (it was news to me that this was possible too!)
  The solution to this is to (use importlib-resources)[https://learn.scientific-python.org/development/patterns/data-files/#how-to-package-data-files]
- (Avoid changing state)[https://learn.scientific-python.org/development/principles/design/#avoid-changing-state]. This a real nice simple example of a good alternative to creating a "wonder class" which does everything for you conveniently, if you correctly call all the methods the right order.
- Some nice words about how (flexibility is not always a good thing)[https://learn.scientific-python.org/development/principles/design/#complexity-is-always-conserved]
- (What to put in documentation)[https://learn.scientific-python.org/development/guides/docs/#what-to-include], with some grounding in the so-called "Diataxis framework" which justifies this structure.
- Specific guidance on how to go about [introducing typing gradually](https://learn.scientific-python.org/development/guides/style/#type-checking) to an existing codebase


