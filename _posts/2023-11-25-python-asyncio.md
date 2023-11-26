---
title: Python asyncio
date: 2023-11-25 22:47 -0800
categories: [programming-language, python]
tags: [asyncio, generator, await, event-loop]
---

# Generator

Follow below peps to understand semantics of generator and async/await. Also,
see https://asyncio-notes.readthedocs.io/en/latest/asyncio-history.html for the
history of asyncio. Another good blog is from
[tenthousandmeters](https://tenthousandmeters.com/blog/python-behind-the-scenes-12-how-asyncawait-works-in-python/).

- PEP 255: define semantics of `yield`. It returns next value or raise
  `StopIteration` exception. A Generator can have `return` statement, but not
  `return sth` because if you want to return sth, you can simply yield it
  before return.
- PEP 342: change `yield` from statement to expression. Also defined `send`
  method such that it can passes value to `yield` expression. `next` is
  equivalent to `send(None)`. It is little tricky to see how send works in the
  first glance. Find a few examples online can quickly grasp the idea.
- PEP 380: subgenerators. `yield from`. Define the syntax of delegating
  generators.
- PEP 492: definition of `async/await`.
  [code](https://github.com/python/cpython/commit/7544508f0245173bff5866aa1598c8f6cce1fc5f)
- PEP 3156 asyncio

TODO: read implementation of pydantic

## How does `await` work

To see how exactly `await` works. we need to get familiar with the `dis`
library first. Checkout [dis.md]({% post_url 2023-11-25-dis-module %}). We will use Python 3.11.2 to do some
experiment. We only take about compilation and execution.

Let's look at an example. It is an async function will calls `await`.

```
from asyncio import sleep
async def f():
    await sleep(0)
```

Here, we only talk about the code generation stage of the compilation process.
The compilation code is
[here](https://github.com/python/cpython/blob/0c37ea9abad2eae146ce117eca0503aaedc96c0f/Python/compile.c#L5862).
The first thing to notice is that the implementation is almost the same as
`YieldFrom_kind` above. They both delegate the job to a macro `ADD_YIELD_FROM`.
I would say that coroutine in Python is almost identical to `yield from`. This
is the Python way of realizing bi-directional communication between a caller
and a coroutine. It is quite different from Golang channels, but fulfills the
same purpose. The outcome bytecode is as follows,

```
  2           0 RETURN_GENERATOR
              2 POP_TOP
              4 RESUME                   0

  3           6 LOAD_GLOBAL              1 (NULL + sleep)
             18 LOAD_CONST               1 (0)
             20 PRECALL                  1
             24 CALL                     1
             34 GET_AWAITABLE            0
             36 LOAD_CONST               0 (None)
        >>   38 SEND                     3 (to 46)
             40 YIELD_VALUE
             42 RESUME                   3
             44 JUMP_BACKWARD_NO_INTERRUPT     4 (to 38)
        >>   46 POP_TOP
             48 LOAD_CONST               0 (None)
```

Let's talk about above sequence step by step.

First, it is a call `GET_AWAITABLE`, which extracts the awaitable object of
`sleep(0)`. If we write it as two lines

```python
x = sleep(0)
await x
```

Then this step returns `x` above, which has type `coroutine`. The relevant code
is
[here](https://github.com/python/cpython/blob/955ba2839bc5875424ae745bfab53e880a9ace49/Python/ceval.c#L2546).
The main logic is line `PyObject *iter = _PyCoro_GetAwaitableIter(iterable);`.
It has two cases:

1. `x` is a coroutine.
2. The `am_await` slot is defined and is a generator. This slot maps to a
   function `__await__` in python code. See
   [code](https://github.com/python/cpython/blob/3ea0beb3599f734bb9387a526dccd5768ad6b1a9/Objects/typeobject.c#L8091).

`sleep(0)` is case#1.
[asyncio.future](https://github.com/python/cpython/blob/981b509784c733c54d47bd4d1ceea34fc7ebcac3/Lib/asyncio/futures.py#L284)
belongs to case#2. Depending on whether this future is done or not, it has a
`yield` step.

One thing to note that this slot function has type `unaryfunc`. Which means in
python code, it has signature `def __await__(self)`. Second thing about
`am_await` is that this function must return a generator.
[\_PyCoro_GetAwaitableIter](https://github.com/python/cpython/blob/836b17c9c3ea313e400e58a75f52b63f96e498bb/Objects/genobject.c#L1058-L1059)
explicitly checks it cannot be a coroutine. I emphasize one more time: it
returns an iterator, not async iterator. This means that `__await__` function's
implementation is like

```python
def __await__(self):
    ...
    yield ...
    ...
    yield ...
    ...
    return ...
```

The implementation of `__await__` may have zero or multiple yield statements.
Which one is returned in the statement `await xxx`? Wait a second, we haven't
get to that step yet. `GET_AWAITABLE` only returns `__await__()`, it hasn't
iterate it yet. `SEND` triggers `__await__` to run. The line above `SEND` is
`LOAD_CONST 0`, which means we are doing `send(None)`. We all know that the
first `send` call must have `None` as the argument, otherwise, you get the
following error

```
TypeError: can't send non-None value to a just-started generator
```

After `send(None)`, the iterator is triggered once. If `__await__` has at least
one `yield` statement, then the stack top contains this yield value. If
`__await__` does not have any `yield` statement, then the stack top contains
the return value of `__await__`. The next command `YIELD_VALUE` is simple. It
returns the stack top to caller and marks the current frame as suspended.

One extra note about `send`. How does `send` drives a generator to run? and how
does Python interpreter knows where is left last time, so it can pick up from
last yield place. From the
[code](https://github.com/python/cpython/blob/955ba2839bc5875424ae745bfab53e880a9ace49/Python/ceval.c#L2592),
you can see that it just calls `tp_iternext` slot. For a generator, it is just
`gen_iternext`. This function will add this generator's frame to the frame list
and execute the generator. Also, `frame.f_lasti` and `frame.f_locals` are kept
in this frame. After this generator's frame is revisited, it can pick up from
where it left last time. This detail is needed to understand how iterator works
in Python, which is a prerequisite to understand async/await. See
[this post](https://www.cnblogs.com/traditional/p/13620438.html) for more
details.

Ignore `RESUME` for now. What follows is
`JUMP_BACKWARD_NO_INTERRUPT 4 (to 38)`. This command unconditionally jumps to
address 38, i.e., SEND. Basically, it calls SEND again and drives `__await__`
to execute code after the first `yield` statement. It is a loop! It keeps
running `SEND` command until there are no more `yeild` statement inside
`__await__`. Wait. Why is it a loop? I thought whenever Cpython interpreter
sees a `yield`, it should return the control to its parent. This is true. See
last paragraph that `gen_iternext` will remove the generator frame from the
frame chain, so it returns the control. So this loop is an interruptive. It
just means that next time we enter this loop again, we pick up where we left.
Wait. When the hell does this loop end? If you read the disassembled code
carefully. You see that `SEND` is also a potential jump command. Check
[dis.md]({% post_url 2023-11-25-dis-module %}) to see why it is a "potential jump". The
[send implementation](https://github.com/python/cpython/blob/955ba2839bc5875424ae745bfab53e880a9ace49/Python/ceval.c#L2621)
says if we encounters a `return` statement instead of a `yield` statement, then
we jump to a location. In our example it is `46 POP_TOP`. At this moment, the
return value will be passed to the left side of `y = await xxx`.

To sum up, `await` keyword accepts a coroutine or an object that has defined
the function `__await__`. This function must be a generator with signature
`def __await__(self)`. It can has zero or many `yield` statements, and its
return value will be passed to the left side of the `await` statement.

## How does event loop work

The core part is
[\_run_once](https://github.com/python/cpython/blob/99da75e770a7108cd046b103ada27cf31427fb0b/Lib/asyncio/base_events.py#L1845)
function. The core member variable is `self._ready`. Asyncio does an epoll wait
and put callbacks for all triggered file descriptor in `self._ready`.
Meanwhile, some utility functions such as `call_soon` directly puts the
callback in `self._ready`, so it will be executed in next cycle.

## asyncio.Future and asyncio.Task

We have two futures. One in the concurrent package. One in the asyncio package.
`asyncio.Future` is awaitable. The `__await__` function is defined as below

```python
def __await__(self):
    if not self.done():
        self._asyncio_future_blocking = True
        yield self  # This tells Task to wait for completion.
    if not self.done():
        raise RuntimeError("await wasn't used with future")
    return self.result()  # May raise too.
```

When the future is not done, it yields itself, otherwise, it returns the
result. This makes total sense. When we first await a future, we hang this
coroutine and run something else, which may call `set_result` or
`set_exception` of this future, and insert the original coroutine to the event
loop, so the await expression returns. Checkout
[wrap_future](https://github.com/python/cpython/blob/981b509784c733c54d47bd4d1ceea34fc7ebcac3/Lib/asyncio/futures.py#L409)
function. It transforms a concurrent future to a asyncio future.

1. For a `concurrent.futures.Future` object, we create an empty
   `asyncio.Future`.
2. Register `asyncio.Future.set_result` as a callback for this
   `concurrent.futures.Future`, such that when the concurrent future is done,
   the asyncio future will be marked as done as well.
3. return the `asyncio.Future`.

This is very cool!. We only need a threadpool to convert a synchronous library
to asynchronous. I found Rockset's Python client library utilizes this
[trick](https://github.com/rockset/rockset-python-client/blob/451299ffba96b8175b2b091b82cf84106f7fdd67/rockset/api_client.py#L42-L43).
We should admit that this trick is not optimal. It is kind of a hacky
workaround if you have a legacy codebase and want to quickly make it asyncio
compatible.

Task is a subclass of asyncio.Future, and they share the same `__await__`
implementation. The difference is that Future is very general. It does not
provide any concrete code that drives the `SEND -> YIELD_VALUE -> SEND -> ..`
cycle. On the other hand, Task is a wrapper of a coroutine. This wrapper layer
or adapter layer makes this coroutine can be executed in an event loop. So, one
of Task's responsibility is to drive the `SEND -> YIELD_VALUE` cycle. This
function is called
[\_\_step](https://github.com/python/cpython/blob/981b509784c733c54d47bd4d1ceea34fc7ebcac3/Lib/asyncio/tasks.py#L250).
This function is long, but it just does two things. It calls `SEND(None)` and
`self._loop.call_soon(self.__step)` in various conditions. A lot of edge cases
makes this function long, but for our purpose, we only care about two cases.
One case is `SEND(None)` returns a Future. The other case is that `SEND(None)`
encounters a `return` statement, i.e., `StopIteration` exception. For the first
case, it calls `self_.loop.call_soon(self.__step)` to enqueue the function to
the event loop. For the second case, the returned Future is done, so it adds a
callback to this future, and this callback will call `self.__step` again.

One thing to note that Task's constructor calls
`self._loop.call_soon(self.__step)`. Which means once a task is created, it
starts running inside the event loop immediately. It does not need to be
awaited to start execution.

Coroutine runs as a task in the asyncio event loop. Checkout `asyncio.run(co)`
implementation. It basically wrap this coroutine `co` into a task, and then
call `run_until_complete`.

## How does `asyncio.gather` work

`asyncio.gather` take a list of coroutines as input and returns a
`_GatheringFuture` object. You may wonder who calls `set_result` or
`set_exception` for this future. Read blow.

Tasks will be created from the input coroutines usingk
`fut = ensure_future(arg, loop=loop)`, and register a callback function. Inside
this callback function, the last finished task will call `outer.set_result` or
`outer.set_exception`. So you see it is like a latch in multithreading.

## patterns learned from graphql

The `graphql-core-legacy` provides good patterns of using asyncio. For example,
below executor.

```python
class AsyncioExecutor(object):
    def __init__(self, loop=None):
        # type: (Optional[AbstractEventLoop]) -> None
        if loop is None:
            loop = get_event_loop()
        self.loop = loop
        self.futures = []  # type: List[Future]

    def wait_until_finished(self):
        # type: () -> None
        # if there are futures to wait for
        while self.futures:
            # wait for the futures to finish
            futures = self.futures
            self.futures = []
            self.loop.run_until_complete(wait(futures))

    def clean(self):
        self.futures = []

    def execute(self, fn, *args, **kwargs):
        # type: (Callable, *Any, **Any) -> Any
        result = fn(*args, **kwargs)
        if isinstance(result, Future) or iscoroutine(result):
            future = ensure_future(result, loop=self.loop)
            self.futures.append(future)
            return Promise.resolve(future)
        elif isasyncgen(result):
            return asyncgen_to_observable(result, loop=self.loop)
        return result
```

# Typing

Async related typing is always fantastic and hard to understand when starting
working with them. For example, given below code, what is the return type of
function `f` and what is the type `x`?

```python
async def f():
    yield 1

async for x in f():
    print(x)
```

## async_generator, typing.AsyncGenerator and collections.abc.AsyncGenerator

Reading the
[typing module documentation](https://docs.python.org/3.12/library/typing.html#typing.AsyncGenerator),
we can see that function `f` above has return type
`typing.AsyncGenerator[int, None]`. Let's verify it!

```
In [114]: async def f():
     ...:     yield 1
     ...:

In [115]: x = f()

In [116]: type(x)
Out[116]: async_generator
```

It is close, but not exactly the type we want. Let's check their relationship.

```
In [119]: type(x) == typing.AsyncGenerator
Out[119]: False

In [123]: issubclass(type(x), typing.AsyncGenerator)
Out[123]: True

In [124]: isinstance(typing.AsyncGenerator, type(x))
Out[124]: False

In [125]: type(x).__mro__
Out[125]: (async_generator, object)
```

The result is super strange. It shows that `async_generator` is a subclass of
`typing.AsyncGenerator`, but the mro of `async_generator` does not contain its
parent class!

To figure our all this non-sense, let's take one step back: where type
`async_generator` is defined? According to Python naming convention, all
classes should be camel case, so this means this type is defined in C code, and
this should be a genuine C type. Ah. It is
[PyAsyncGen_Type](https://github.com/python/cpython/blob/836b17c9c3ea313e400e58a75f52b63f96e498bb/Objects/genobject.c#L1578)

```
PyTypeObject PyAsyncGen_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "async_generator",                          /* tp_name */
    ...
```

OK. It becomes even more strange. How can a C type `async_generator` inherit a
python type `typing.AsyncGenerator`? Shouldn't C types be the most fundamental
types? In order to answer this question, we need to introduce another type
`collections.abc.AsyncGenerator`.

First, `typing.AsyncGenerator` is defined as a wrapper on top of
`collections.abc.AsyncGenerator`. See
[code](https://github.com/python/cpython/blob/29ff9daf823ec7af7875c6642f1e191ed48e3b73/Lib/typing.py#L2698).
`AsyncGenerator = _alias(collections.abc.AsyncGenerator, 2)`. This `_alias` is
`class _SpecialGenericAlias`. Basically, `typing.AsyncGenerator` is an instance
of `_SpecialGenericAlias`. Its constructor takes two parameters `origin` and
`nparams`. Here, `nparams` means this type should take two subscriptions such
as `typing.AsyncGenerator[int, None]` when used.

```
In [131]: typing.AsyncGenerator.__origin__
Out[131]: collections.abc.AsyncGenerator

In [132]: typing.AsyncGenerator._nparams
Out[132]: 2
```

Also, `_SpecialGenericAlias` provides its own implementation of
`__subclasscheck__`, so `collections.abc.AsyncGenerator` is a subclass of
`typing.AsyncGenerator`. See [pep-3119](https://peps.python.org/pep-3119/) for
more details.

```
In [130]: issubclass(collections.abc.AsyncGenerator, typing.AsyncGenerator)
Out[130]: True
```

Second, `async_generator` is registered as a subclass of
`collections.abc.AsyncGenerator`. The code is
[AsyncGenerator.register(async_generator)](https://github.com/python/cpython/blob/59a99ae277e7d9f47edd4a538c1239d39f10db0c/Lib/_collections_abc.py#L249-L250)
This is how you can make a C type become a subtype of a Python type. Because C
code cannot be changed, thus `ABCMeta.register` is the only way to go. I was
very confused the first time I see function `ABCMeta.register` because I
thought who the hell will use this function to register a class as a subclass
of another class. Why not just define it as `class Derived(Base)`? Simple and
elegant, right? Another use case of `ABCMeta.register` is that if the derived
class is provided by a 3rd party library, and you cannot change it. This is the
way to go.

OK. So far we have all the pieces needed to understand the type hierarchy here.

```
subclass chain (from base to derived): typing.AsyncGenerator -> collections.abc.AsyncGenerator -> async_generator
```

Thus we can annotate our examples as follows

```
async def f() -> typing.AsyncGenerator[int, None]:
    yield 1
```

This is not the end of the story. If you read the
[typing module documentation](https://docs.python.org/3.12/library/typing.html#typing.AsyncGenerator)
carefully, you see a line

```
Deprecated since version 3.9: collections.abc.AsyncGenerator now supports subscripting ([]). See PEP 585 and Generic Alias Type.
```

Basically, we no longer need this auxiliary variable `typing.AsyncGenerator`
any more. We can use `collections.abc.AsyncGenerator` to annotate directly. How
is it implemented? See
[this line](https://github.com/python/cpython/blob/59a99ae277e7d9f47edd4a538c1239d39f10db0c/Lib/_collections_abc.py#L180).
`collections.abc.AsyncIterable` defines `__class_getitem__`, so we can write
`AsyncIterable[int, float]`. See more details in
[pep-560](https://peps.python.org/pep-0560/). However, I find the current
implementation is not usable for typing purpose. The current implementation
does not provide any validation, so I can provide more than two arguments.
While `typing.AsyncGenerator` only accept two subscriptions.

```
Out[135]: collections.abc.AsyncGenerator[int, int, int, float, float]

In [136]: typing.AsyncGenerator[int, int, int]
---------------------------------------------------------------------------
TypeError: Too many arguments for typing.AsyncGenerator; actual 3, expected 2
```
