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

gdb supports python API after gdb-7.0, but it must be configured with
`--with-python`. We can use `gdb --configuration` to check if python API is
supported or not. See more details of
[this post](https://sourceware.org/pipermail/gdb/2015-April/045235.html).

Using ubuntu as an example, you only need to `apt-get install gdb pytohn3-dbg`,
and then you can attach to a process `gdb python <pid>` to debug a running
python program. However, for my case, our program runs using python3.8, and the
system python version is python3.9. so `apt-get install python3-dbg` actually
installs `python3.9-dbg`. It cannot find the python stacktrace as blow:

```
(gdb) py-bt
Unable to locate python frame
```

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

## class

[Python grammar](https://docs.python.org/3/reference/grammar.html) has class
definition as

```
class_def:
    | decorators class_def_raw
    | class_def_raw

class_def_raw:
    | 'class' NAME ['(' [arguments] ')' ] ':' block

rguments:
    | args [','] &')'

args:
    | ','.(starred_expression | ( assignment_expression | expression !':=') !'=')+ [',' kwargs ]
    | kwargs
```

So basically, class can be defined as

```
class A(B, C, k1=v1, k2=v2):
    ...
```

What do these `B`, `C`, `k1` and `k2` could be? This is answered inside
function
[builtin**\_build_class**](https://github.com/python/cpython/blob/f3909b8bc83675b7ab093dbc558e677558d8e718/Python/bltinmodule.c#L91).
Basically, `B` and `C` should be subclasses. `k1` and `k2` could define
metaclass or class namespace variables.

### Reference

- https://eli.thegreenplace.net/2012/06/15/under-the-hood-of-python-class-definitions

## Refs

- https://tenthousandmeters.com/
