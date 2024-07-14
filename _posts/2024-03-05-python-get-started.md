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

I tried to use the `cpython-lldb` package, but I do not think it works for the
latest version of Cpython as I encountered wired behavior when printing
dictionaries. If you want to try it out, you need to get familiar with the
script feature of lldb. First, we need to know which version of python it links
to.

```
$ otool -L /Applications/Xcode.app//Contents/SharedFrameworks/LLDB.framework/LLDB | grep -i pytho
        @rpath/Python3.framework/Versions/3.9/Python3 (compatibility version 3.9.0, current version 3.9.0)
        @rpath/Python3.framework/Versions/3.9/Python3 (compatibility version 3.9.0, current version 3.9.0)
```

We cannot use a different version of python.

However, I realize that I do not need to get into the python script in most
cases. The `expr` command is powerful enough to inspect the local state.

The example python code to debug is

```python
class A:
    def __init__(self, x = 5, y = 10):
        self.x = x
        self.y = y

    def foo(self):
        print(self.x)

    def __repr__(self):
        return f"<A>({self.x}, {self.y})"


a = A(5)
a.x
```

Example 1: print `repr(obj)`

```
(lldb) expr PyUnicode_AsUTF8(v->ob_type->tp_repr(v))
(const char *) $18 = 0x00000001009bf430 "<A>(5, 10)"
```

Example 2: print `obj.__dict__`

```
(lldb) expr -- PyObject* $xx = PyObject_GenericGetDict(v, 0)
(lldb) expr -- PyObject* $yy = ($xx)->ob_type->tp_repr($xx)
(lldb) expr PyUnicode_AsUTF8($yy)
(const char *) $2 = 0x00000001009bf3d0 "{'x': 5, 'y': 10}"
```

Example 3: get attribute

```
(lldb) expr -- PyObject* $rr = PyObject_GetAttr(v, PyUnicode_FromString("y"))
(lldb) expr PyUnicode_AsUTF8($rr->ob_type->tp_repr($rr))
(const char *) $21 = 0x000000010097b4f0 "10"
```

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

or

```bash
export PYTHONBREAKPOINT=IPython.embed
```

if IPython is installed.

For pytest, you can use below command to automatically drop into ipython shell.

```
pytest --pdbcls=IPython.terminal.debugger:TerminalPdb
```

For multithreading programs, neither pdb or `IPython.embed` works well. Suppose
you add a breakpoint at the same location in 3 threads, then once the
breakpoint is hit, 3 pdb instances are created, and when you type some command
in one pdb console and hit enter, it jumps to the pdb instance in another
thread, which is very confusing. `Ipython.embed` is worse in this case. It
throws exception, gets stuck and needs `kill -9` to stop it. The community has
been asking for stopping all threads behavior for a long time. See this
[stackoverflow answer](https://stackoverflow.com/a/64678235/3183330).

One way to walk around this issue is using a global thread lock.

```python
import threading
_lock = threading.Lock()
...

with _lock:
  breakpoint()
...
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

Pyrasite's implementation is interesting. Basically, it uses
`PyRun_SimpleString`.

```
(gdb) call PyGILState_Ensure()
$1 = PyGILState_UNLOCKED
(gdb) call PyRun_SimpleString("print(5+10); print(1000)")
(gdb) set $f = (FILE*)(fopen("xiong2.py", "r"))
(gdb) print $f
$19 = (FILE *) 0x5630fc64f370
(gdb) call PyRun_SimpleFile($f, "xiong2.py")
(gdb) call fclose($f)
(gdb) call PyGILState_Release(1)
```

See value of
https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Include/pystate.h#L77

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
