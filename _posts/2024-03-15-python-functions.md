---
layout: post
title: Python -- functions
date: 2024-03-15 11:57 -0700
categories: [programming-language, python]
tags: [python]
---

Function sounds simple, but its implementation is quite complicated inside
cpython. There are functions in the global space, functions inside a class
(i.e., method), class method, static method, and etc.

## Function lookup

Let's test your understanding with a few examples.

```
In [5]: class A:
   ...:     def f(self): ...
   ...:

In [6]: A.f
Out[6]: <function __main__.A.f(self)>

In [7]: A().f
Out[7]: <bound method A.f of <__main__.A object at 0x10736c450>>

In [8]: import inspect

In [9]: inspect.getattr_static(A(), 'f')
Out[9]: <function __main__.A.f(self)>
```

In the above example, why do `A.f` and `inspect.getattr_static(A(), 'f')`
returns a function object, but `A().f` returns a bound method?

Let's take a look at `A().f` and `A.f` first. Both attribute lookups will
generate opcode
[LOAD_ATTR](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Python/ceval.c#L3460)
which calls function
[PyObject_GetAttr](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Objects/object.c#L904).
This function is simple. It invokes the `tp_getattro` method on the current
object type. For `A().f` the object type is a newly defined class `A` and its
`tp_getattro` is set to `PyObject_GenericGetAttr`. For `A.f`, the class type is
`type` and `tp_getattro` is set to
[type_getattro](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Objects/typeobject.c#L4410).

- For `PyObject_GenericGetAttr`, the core code is
  [here](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Objects/object.c#L1271-1283).
  It first looks up the attribute by the attribute name `f`, which returns a
  function object. Then it checks whether `descr` has defined the slot
  `tp_descr_get`. What is this slot? We are all familiar with the `@property`
  annotation in Python. Underneath, it is this`tp_descr_get` slot. It means the
  attribute has customized `__get__` and `__set__` methods. So is a function a
  descriptor? Yes. The definition of a function is
  [here](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Objects/funcobject.c#L757).
  You see that `tp_descr_get` member is set, which means it is a descriptor.

- For `type_getattro`, the core code is
  [here](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Objects/typeobject.c#L3946-L3960).
  Here input `type` is `A`. The lookup returns the same function object as
  `PyObject_GenericGetAttr`. However, the biggest difference is that here we
  call this `tp_descr_get` slot with `NULL` for the second parameter. Below is
  the code of this slot. When you pass an non-null object to it, it returns you
  a `method`. Otherwise, it returns the function directly.

  ```python
  /* Bind a function to an object */
  static PyObject *
  func_descr_get(PyObject *func, PyObject *obj, PyObject *type)
  {
      if (obj == Py_None || obj == NULL) {
          Py_INCREF(func);
          return func;
      }
      return PyMethod_New(func, obj);
  }
  ```

OK. Everything makes sense. To sum up, the function object is descriptor, and
depending on when it is called, it can return the function directly or wrap it
inside a bound method.

One additional node about `PyObject_GetAttr` function. You can see that it
checks `tp_getattro` slot first. If it does not exist, then it checks
`tp_getattr` slot. Actually, both slots serve the same purpose, but
`tp_getattr` is deprecated. See the
[official documentation](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Doc/c-api/typeobj.rst#L759).

Finally, let's explain `inspect.getattr_static(A(), 'f')`. The
[documentation](https://docs.python.org/3/library/inspect.html#inspect.getattr_static)
says it well

> getattr_static() does not resolve descriptors, for example slot descriptors
> or getset descriptors on objects implemented in C. The descriptor object is
> returned instead of the underlying attribute.

Basically, it does not invoke the descriptor logic. It simply returns the
dictionary lookup. The implementation of `getattr_static` is approximately
shown below.

```
In [16]: type(A()).__dict__['f']
Out[16]: <function __main__.A.f(self)>
```

## Function execution

Again, let's start with a simple example.

```python
def foo(x):
    print(x)

class A:
    f = foo

A().f(5)
```

Above code does not work

```
Traceback (most recent call last):
  File "/Users/xiongding/tmp/test2.py", line 8, in <module>
    A().f(5)
TypeError: foo() takes 1 positional argument but 2 were given
```

The fix is simple, you just need to change the signature of `foo` to
`foo(self, x)`. Probably everyone knows the fact that a bound method implicitly
takes the calling object as the first argument, so we need a `self` parameter
when defining this function. Meanwhile for plain functions, we do not expect a
`self` argument. How does Cpython decide when to insert this `self` argument or
not?

The byte codes of above python code is below.

```
  0           0 RESUME                   0

  2           2 LOAD_CONST               0 (<code object foo at 0x100696f70, file "test2.py", line 2>)
              4 MAKE_FUNCTION            0
              6 STORE_NAME               0 (foo)

  5           8 PUSH_NULL
             10 LOAD_BUILD_CLASS
             12 LOAD_CONST               1 (<code object A at 0x100829b00, file "test2.py", line 5>)
             14 MAKE_FUNCTION            0
             16 LOAD_CONST               2 ('A')
             18 PRECALL                  2
             22 CALL                     2
             32 STORE_NAME               1 (A)

  8          34 PUSH_NULL
             36 LOAD_NAME                1 (A)
             38 PRECALL                  0
             42 CALL                     0
             52 LOAD_METHOD              2 (f)
             74 LOAD_CONST               3 (5)
             76 PRECALL                  1
             80 CALL                     1
             90 POP_TOP
             92 LOAD_CONST               4 (None)
             94 RETURN_VALUE

  ...
```

The most important part is
[LOAD_METHOD](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Python/ceval.c#L4483)
Depending on the function is bound method or not, the stack layout is
different.

```
bound method case: meth | self | arg1 | ... | argN
       other case: NULL | meth | arg1 | ... | argN
```

The `NULL` element in the stack tells whether the method is bounded or not. So
what is inside
[\_PyObject_GetMethod](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Objects/object.c#L1150)?
Using above example, let's analyze different cases.

- Case: `A().f(5)`

  In this case, `tp` is `A`, so `tp->tp_getattro != PyObject_GenericGetAttr`
  does not hold. `PyObject *descr = _PyType_Lookup(tp, name);` return the
  function object `f`. This is unbounded function, and it should have flag
  `Py_TPFLAGS_METHOD_DESCRIPTOR`. So ends up with below case

  ```
  if (_PyType_HasFeature(Py_TYPE(descr), Py_TPFLAGS_METHOD_DESCRIPTOR)) {
      meth_found = 1;
  ```

- Case: `A.f(5)`

  In this case, `tp` is `type`, so `tp->tp_getattro = type_getattro` and thus
  `tp->tp_getattro != PyObject_GenericGetAttr` holds. So we end up with his
  case

  ```python
  if (tp->tp_getattro != PyObject_GenericGetAttr || !PyUnicode_CheckExact(name)) {
      *method = PyObject_GetAttr(obj, name);
      return 0;
  }
  ```

Let's make some change to above code

```
def foo(x):
    print(x)
class A:
    f = staticmethod(foo)
```

- Case: `A().f(5)`

  Similar to the non-static case, we get the function object `descr`, but this
  time it does not have flag `Py_TPFLAGS_METHOD_DESCRIPTOR`. So it get to line
  `f = Py_TYPE(descr)->tp_descr_get;`. staticmethod is also a descriptor! Th
  definition is
  [here](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Objects/funcobject.c#L1213).
  Finally, it ends up at below case.

  ```python
  if (f != NULL) {
      *method = f(descr, obj, (PyObject *)Py_TYPE(obj)); // label c
      Py_DECREF(descr);
      return 0;
  }
  ```

- Case: `A.f(5)`

  This is the exact same as the non-static case.

There are so many details above. To sum up in one sentence: cpython correctly
detects whether a function is a bound method or not.

Also, one more note about `staticmethod`. Its `tp_descr_get` function is
defined
[here](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Objects/funcobject.c#L1089).
Other than a function, it always return a pure function. Also, no matter it is
called from `A` or `A()`, it is always detected as non-bound function. On the
contrary, `classmethod`'s `tp_descr_get` always returns a bound method.
