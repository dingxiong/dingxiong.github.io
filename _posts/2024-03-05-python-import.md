---
layout: post
title: Python -- import
date: 2024-03-05 14:49 -0800
categories: [programming-language, python]
tags: [python]
---

## How does import work in Python

Below findings are based on Cpython version of
`6066739ff7794e54c98c08b953a699cbc961cd28`.

## Bootstrap

This step generates two C files `Python/frozen.c` and
`Python/deepfreeze/deepfreeze.c` from `importlib` python files. See part of
Makefile below

```Makefile
############################################################################
# frozen modules (including importlib)
#
# Freezing is a multi step process. It works differently for standard builds
# and cross builds. Standard builds use Programs/_freeze_module and
# _bootstrap_python for freezing and deepfreezing, so users can build Python
# without an existing Python installation. Cross builds cannot execute
# compiled binaries and therefore rely on an external build Python
# interpreter. The build interpreter must have same version and same bytecode
# as the host (target) binary.
#
# Standard build process:
# 1) compile minimal core objects for Py_Compile*() and PyMarshal_Write*().
# 2) build Programs/_freeze_module binary.
# 3) create frozen module headers for importlib and getpath.
# 4) build _bootstrap_python binary.
# 5) create remaining frozen module headers with
#    ``./_bootstrap_python Programs/_freeze_module.py``. The pure Python
#    script is used to test the cross compile code path.
# 6) deepfreeze modules with _bootstrap_python
#
# Cross compile process:
# 1) create all frozen module headers with external build Python and
#    Programs/_freeze_module.py script.
# 2) deepfreeze modules with external build Python.
#
```

Steps as follow

1. Compile `Programs/_freeze_module.c` to an executable `_freeze_module`. This
   executable compiles python code to byte code.
2. Call `_freeze_module` with all freeze-in files, namely, the
   `FROZEN_FILES_IN` list. The output files are `FROZEN_FILES_OUT`. For
   example,
   ```bash
   ./_freeze_module importlib._bootstrap $(srcdir)/Lib/importlib/_bootstrap.py Python/frozen_modules/importlib._bootstrap.h
   ```
   means generating `importlib._bootstrap.h` from `_bootstrap.py`. The output
   file contains the corresponding compiled byte code of input file.
3. Call `freeze_module.py` on `FROZEN_FILES_IN` to generate `Python/frozen.c`.
4. Call `deppfreeze.py` on `FROZEN_FILES_OUT` to generate
   `Python/deepfreeze/deepfreeze.c`.

## Registration

`__import__` is a builtin module. It is registered during Python interpreter
initialization stage. See below code snippet.

```C
// bltinmodule.c.h
#define BUILTIN___IMPORT___METHODDEF    \
    {"__import__", (PyCFunction)(void(*)(void))builtin___import__, METH_FASTCALL|METH_KEYWORDS, builtin___import____doc__},

```

It is assigned to Python interpreter state `interp->import_func` in function
`pylifecycle.c#pycore_init_builtins`.

```C
// Get the __import__ function
PyObject *import_func = _PyDict_GetItemStringWithError(interp->builtins,
                                                        "__import__");
if (import_func == NULL) {
    goto error;
}
interp->import_func = Py_NewRef(import_func);
```

The actual implementation call diagram is as blow.

```
builtin___import__
    -> PyImport_ImportModuleLevelObject
        -> mod = import_find_and_load(tstate, abs_name)
            -> mod = PyObject_CallMethodObjArgs(interp->importlib, &_Py_ID(_find_and_load),abs_name, interp->import_func, NULL);
```

So finally, it calls `_find_and_load` method in `Lib/importlib/_bootstrap.py`.
How? How could C code call a python file? As said in the **Bootstrap** section,
this python file is compiled to byte code and saved in a C file. See below
[Initialization](##Initialization) section to understand how Python loads this
byte code when interpreter starts. Note, registration happens before below
[Initialization](##Initialization) step. But it does not hurt because it is
just registration. `__import__` is not called yet.

Also, in above call stack, Python will transform relative import path or
absolute import path, and takes different strategies for syntax `import a.b.c`
and `from a.b import c`.

## Initialization.

`importlib` is initialized when python interpreter is initialized. There are a
few steps.

1. The first step is to load the frozen module. The main function is
   `import.c#PyImport_ImportFrozenModule("_frozen_importlib")`. It first finds
   the module information for `_frozen_importlib`, which is inside `frozen.c`.
   More precisely, `frozen.c` borrows
   `deepfreeze.c#_Py_get_importlib__bootstrap_toplevel` for loading the frozen
   byte code. Call sequence:
   `pylifecycle.c#init_importlib -> import.c#PyImport_ImportFrozenModule("_frozen_importlib") -> find_frozen("_frozen_importlib", &info) -> unmarshal_frozen_code(&info) -> d = module_dict_for_exec(tstate, name); // get the dictionary associated with _frozen_importlib module -> m = import_add_module(tstate, name) // not found "_frozen_importlib" module, so create a new module, // and add to "tstate->interp->modules" // The newly creately "_frozen_importlib" will not have builtins functions, // so we after create it, we also add builtins to its "md_dict" -> m = exec_code_in_module(tstate, name, d, co); -> v = PyEval_EvalCode(code_object, module_dict, module_dict); // run the frozen byte code, so get the _frozen_importlib -> m = import_get_module(tstate, name)`
2. After `_frozen_importlib` module is found, it is added to
   `tstate->interp->modules`. Also `interp->imporlib` is set to the loaded
   `_frozen_importlib`.
3. Bootstrap `_imp` module. (TODO: study in detail)
4. call `importlib._install` method.

## Execution

When Python sees `import a.b.c`, it translates it to byte code `IMPORT_NAME` in
`ceval.c`, which calls `import_name`

```C
static PyObject *
import_name(PyThreadState *tstate, _PyInterpreterFrame *frame,
            PyObject *name, PyObject *fromlist, PyObject *level)
{
    PyObject *import_func, *res;
    PyObject* stack[5];

    import_func = _PyDict_GetItemWithError(frame->f_builtins, &_Py_ID(__import__));
    if (import_func == NULL) {
        if (!_PyErr_Occurred(tstate)) {
            _PyErr_SetString(tstate, PyExc_ImportError, "__import__ not found");
        }
        return NULL;
    }
    PyObject *locals = frame->f_locals;
    /* Fast path for not overloaded __import__. */
    if (import_func == tstate->interp->import_func) {
        int ilevel = _PyLong_AsInt(level);
        if (ilevel == -1 && _PyErr_Occurred(tstate)) {
            return NULL;
        }
        res = PyImport_ImportModuleLevelObject(
                        name,
                        frame->f_globals,
                        locals == NULL ? Py_None :locals,
                        fromlist,
                        ilevel);
        return res;
    }

    Py_INCREF(import_func);

    stack[0] = name;
    stack[1] = frame->f_globals;
    stack[2] = locals == NULL ? Py_None : locals;
    stack[3] = fromlist;
    stack[4] = level;
    res = _PyObject_FastCall(import_func, stack, 5);
    Py_DECREF(import_func);
    return res;
}
```

You can see that basically, it uses `__import__` method and assigned it to
variable `import_func`. Then it compares it with `tstate->interp->import_func`.
In [Registration](##Registration) section, we see that `interp->import_func` is
assigned to `__import__`, then it seems that this comparison should always be
true. But user may install import hooks, or even override `__import__`
function. (TODO: verify this statement).

## Summary

After stating so much about import mechanism in Python, now we know that
`Lib/importlib/_bootstrap.py` is the place to find all detailed logic.

## References

- http://blog.tpleyer.de/posts/2017-04-02-A-closer-look-at-Pythons-import-mechanism.html
- https://tenthousandmeters.com/blog/python-behind-the-scenes-11-how-the-python-import-system-works/
