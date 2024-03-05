---
layout: post
title: Python -- Get started
date: 2024-03-05 14:28 -0800
categories: [programming-language, python]
tags: [python]
---

## cpython

### Compile

On a 2019 Macbook Pro.

```
MACOSX_DEPLOYMENT_TARGET=10.15 ./configure --with-pydebug
bear -- make -j -s
```

On a 2022 Macbook M1. I have some problem with openssl. So I follow
[this post](https://bugs.python.org/issue40840) to resolve this issue.

```
./configure --with-pydebug --with-openssl=$(brew --prefix openssl)
bear -- make -j -s
```

Note, I use `bear` to generated compilation database.

### Gdb debug

https://hackmd.io/@klouielu/ByMHBMjFe?type=view

- 1. sudo gdb ./python.exe
- 2. `set startup-with-shell off`

## conda, conda-forge and miniforge

conda is a pacakge manager. While conda-forge is a channel. Conda's default
channel will charge you in some commercial use cases

miniforge-installed conda is the same as Miniconda-installed conda, except that
it uses the conda-forge channel (and only the conda-forge channel) as the
default channel.

## debug

- one way

```
import code; code.interact(local={**globals(), **locals()})
```

one tip: the native python shell does not print variables nicely. Use pprint if
it is available `from pprint import pprint as p`. Or, if ipython is available,
then use below

```
import IPython; IPython.embed()
```

- pdb See PEP 553

```
breakpoint()
```

- pytest support ipdb

  ```
  pytest --pdbcls=IPython.terminal.debugger:TerminalPdb
  ```

- Monkey patch Sometimes, you want to add more logs to 3rd party library. The
  easiest way is to find the logging level switch for this library to output
  debug logs. However, in cases debug log is not enough and you need to add
  more customized logs, you can use Inheritance or monkey patch. I personally
  find monkey patch is faster in a tryout stage.

- Attach to PID See
  [pyrasite](https://pyrasite.readthedocs.io/en/latest/Shell.html)

## Typing

A trick to get correct subclass type in the base class is `Generic[T]`.

## name mangling

Check
[this](https://stackoverflow.com/questions/62599884/werkzeugs-localproxy-local-where-is-it-initialized)

## Refs

- https://tenthousandmeters.com/
