---
layout: post
title: Python -- Get started
date: 2024-03-05 14:28 -0800
categories: [programming-language, python]
tags: [python]
---

## Cpython

### Build

On a 2019 Macbook Pro.

```bash
MACOSX_DEPLOYMENT_TARGET=10.15 ./configure --with-pydebug
bear -- make -j -s
```

On a 2022 Macbook M1. I have some problem with openssl. So I follow
[this post](https://bugs.python.org/issue40840) to resolve this issue.

```bash
CFLAGS="-O0" ./configure --with-pydebug --with-openssl=$(brew --prefix openssl)
bear -- make -j -s
```

I explicitly specified `-O0` to disable optimization because I found
`--with-pydebug` only enables debugging but does not disable optimization. I
use `bear` to generated compilation database. Note, only use `bear` for the
initial build. For incremental build, `bear` will replace everything inside the
compilation database with the only incremental result. So just do `make` for
incremental build.

### Run cpython inside gdb

Follow this post <https://hackmd.io/@klouielu/ByMHBMjFe?type=view> to get
started with using gdb debugging Cpython.

- `sudo gdb ./python.exe`
- `set startup-with-shell off`

GDB is not available in Macbook M1, so I use lldb instead.

```bash
lldb -- ./python.exe ~/tmp/test.py
```

When using the script feature of lldb, we need to know which version of python
it links to.

```
$ otool -L /Applications/Xcode.app//Contents/SharedFrameworks/LLDB.framework/LLDB | grep -i pytho
        @rpath/Python3.framework/Versions/3.9/Python3 (compatibility version 3.9.0, current version 3.9.0)
        @rpath/Python3.framework/Versions/3.9/Python3 (compatibility version 3.9.0, current version 3.9.0)
```

This means that it links to the system version installed by Xcode. To install
the package `cpython-lldb`, we should do

TODO:

1. how to print a variable's **dict**
2. how to print a variable's type
3. printout common data structures: dict, set, list, number, string

## Debug

There are multiple ways to set breakpoints in Python. PEP 553 introduces
function `breakpoint()`. For older versions, you can do

```python
import code; code.interact(local={**globals(), **locals()})
```

or

```python
import IPython; IPython.embed()
```

if IPython is installed.

For pytest, you can use below command to automatically drop into ipython shell.

```
pytest --pdbcls=IPython.terminal.debugger:TerminalPdb
```

### Attach to a python process

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

Also,
[pyrasite](https://gist.github.com/dingxiong/637010c102e77d3d6db2a641147d5121)
is super helpful to attach to a python process as well.

## Python syntax

### Class

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

### Name mangling

Check out
[this](https://stackoverflow.com/questions/62599884/werkzeugs-localproxy-local-where-is-it-initialized)

## Tools

### pyright

`pyright` is a python language server. It is also a command line tool for type
check.

In order to make it work with python virtualenv. You need to add below configs.

```
"venvPath": "/opt/homebrew/Caskroom/miniforge/base/envs/",
"venv": "website-py3_11_0",
"extraPaths": ["./gen-py/"],
```

The `venvPath` config above can be replaced using command line argument
`-v /opt/homebrew/Caskroom/miniforge/base/envs`.

## Misc

### conda, conda-forge and miniforge

conda is a pacakge manager. While conda-forge is a channel. Conda's default
channel will charge you in some commercial use cases

miniforge-installed conda is the same as Miniconda-installed conda, except that
it uses the conda-forge channel (and only the conda-forge channel) as the
default channel.

## Refs

- <https://tenthousandmeters.com/>
