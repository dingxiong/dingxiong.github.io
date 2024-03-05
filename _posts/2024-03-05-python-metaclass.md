---
layout: post
title: Python -- Metaclass
date: 2024-03-05 14:32 -0800
categories: [programming-language, python]
tags: [python]
---

- PEP 3119

## How does metaclass work?

Read `Python/bltinmodule.c#builtin___build_class__` You can see that the
execute sequence is

1. Class body code block runs, which populates the namespace `ns` of this class
2. Run class's meta class with `ns` to generate the class definition.
   1. It will call Meta class's `__new__`, i.e., `type_new` function to
      generates the new class

### Example

```
class MyMeta(type):
    def __new__(cls, name, bases, ns):
        x = super().__new__(cls, name, bases, ns)
        print("inside my meta", ns)
        return x

class A(metaclass=MyMeta):
    a = 100
    print("initialization a", a)

    def f1():
        print("inside f1")
```

Output of above code is

```
xiongding ~/00git/cpython $ ./python.exe ~/test.py
initialization a 100
inside my meta {'__module__': '__main__', '__qualname__': 'A', 'a': 100, 'f1': <function A.f1 at 0x1011a0940>}
```

## How does AbstractMethod work?

abstractmethod decorator will add a flag indicating that this function is a
abstract method. See
[below code](https://github.com/python/cpython/blob/580fbb018fd0844806119614d752b41fc69660f9/Lib/abc.py#L7).

```
def abstractmethod(funcobj):
   funcobj.__isabstractmethod__ = True
    return funcobj
```

Then, `ABCMeta` records all abstract methods. See
[below code](https://github.com/python/cpython/blob/580fbb018fd0844806119614d752b41fc69660f9/Lib/_py_abc.py#L35)

```
class ABCMeta(type):
    def __new__(mcls, name, bases, namespace, /, **kwargs):
        cls = super().__new__(mcls, name, bases, namespace, **kwargs)
        # Compute set of abstract method names
        abstracts = {name
                     for name, value in namespace.items()
                     if getattr(value, "__isabstractmethod__", False)}
        for base in bases:
            for name in getattr(base, "__abstractmethods__", set()):
                value = getattr(cls, name, None)
                if getattr(value, "__isabstractmethod__", False):
                    abstracts.add(name)
        cls.__abstractmethods__ = frozenset(abstracts)
        ...
```

When you initialize a class and this class has non-empty abstract methods, then
error is raise. See `type_new`
[code](https://github.com/python/cpython/blob/580fbb018fd0844806119614d752b41fc69660f9/Objects/typeobject.c#L3780).
When you subclass a abstract class and override the abstract methods, then see
above in `ABCMeta`, `namespace` only contains the overrided method, so this
subclass won't be abstract.

Example

```
In [19]: from abc import ABCMeta, abstractmethod
    ...:
    ...: class M(ABCMeta):
    ...:     def __new__(cls, name, bases, ns):
    ...:         x = super().__new__(cls, name, bases, ns)
    ...:         print(ns)
    ...:         return x
    ...:
    ...:
    ...: class A(metaclass=M):
    ...:     @abstractmethod
    ...:     def fx(self):
    ...:         pass
    ...:
    ...: class B(A):
    ...:     def fx(self):
    ...:         print('fx')
    ...:
{'__module__': '__main__', '__qualname__': 'A', 'fx': <function A.fx at 0x102ebe790>}
{'__module__': '__main__', '__qualname__': 'B', 'fx': <function B.fx at 0x102ebe940>}

In [20]: A()
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-20-6234893e030b> in <module>
----> 1 A()

TypeError: Can't instantiate abstract class A with abstract methods fx

In [21]: B()
Out[21]: <__main__.B at 0x102fe1460>
```

## How does `__init_subclass__` work?

It is subclass registration. See function `typeobject.c#type_new_impl`.
