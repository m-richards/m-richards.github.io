---
title: "Useful things in python's platform module"
date: 2025-02-09T10:42:37+11:00
draft: false
---

# Useful things in python's `platform` module

Recently I was helping a colleague try and debug a failing installation of GeoPandas. Or to be more precise,
installing Pyogrio, the default IO engine was failing. Now once upon a time this was regularly a pain point due to the
lack of widely available windows wheels but today that's a mostly solved problem. User error seemed a more likely scenario. 

Pretty quickly I was able
to work out that the issue came pypi trying to install via source, and the absence of a C compiler and the required flags
to build GDAL was the error being surfaced. But why was pip installing a source distribution? We eventually
tried downloading the wheel and installing that directly, and found out the platform was incompatible.

The hunch then was that they'd installed 32 bit python. But I struggled to find an easy way to prove that.
Now this week, looking for something else entirely I stumbled across `platform.architecture()` which is exactly what I needed.
the platform module also has a few other useful goodies. Here are some examples on windows:
```python
import platform
platform.node()
'<machine name I am using>'
platform.architecture()
('64bit', 'WindowsPE')
platform.processor()
'Intel64 Family 6 Model 151 Stepping 2, GenuineIntel'
platform.python_version()
'3.10.10'
platform.python_compiler()
'MSC v.1934 64 bit (AMD64)'

platform.release()
'10' # I'm on windows 11, so that's interesting
platform.uname()
uname_result(system='Windows', node='<machine name I am using>', release='10', version='10.0.26100', machine='AMD64')
platform.python_implementation()
'CPython'
```

And on the version of WSL I have sitting around

```python
>>> platform.node()
'<machine name I am using>'
>>> platform.architecture()
('64bit', 'ELF')
>>> platform.processor()
'x86_64'
>>> platform.python_version()
'3.10.12'
>>> platform.python_compiler()
'GCC 11.4.0'
>>> platform.release()
'5.15.167.4-microsoft-standard-WSL2'
>>> platform.uname()
uname_result(system='Linux', node='<machine name I am using>', release='5.15.167.4-microsoft-standard-WSL2', version='#1 SMP Tue Nov 5 00:21:55 UTC 2024', machine='x86_64')
>>> platform.python_implementation()
'CPython'
>>>
```

