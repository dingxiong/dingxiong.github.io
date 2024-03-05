---
layout: post
title: Python -- Pytest
date: 2024-03-05 14:48 -0800
categories: [programming-language, python]
tags: [python, pytest]
---

## Basic flow

Entry point is `src/pytest/__main__.py -> src/_pytest/config/__init__.py:main`.
Then it creates a `Config` object, which is a core concept in Pytest. `Config`
initializes a plugin manager. See blow [Pytest plugins](#pytest-plugins)
section.

After initialization finishes, `Config` calls the hook implementation
`pytest_cmdline_main` as below.

```
ret: Union[ExitCode, int] = config.hook.pytest_cmdline_main(
    config=config
)
```

Which invokes the configure hooks and runtest_mainloop.

## pytest-mock

[pytest-mock](https://github.com/pytest-dev/pytest-mock) is a pytest plugin
that provides a fixture `mocker` that simplifies the patch annotation, so you
do not need to remember that correspondence between patch order and argument
order.

- `mock_calls` returns call object and can be expanded as a tuple
  `(name, args, kwargs)`.

## pytest and unittest

pytest fixture and unittest patch order is important. See
https://stackoverflow.com/questions/25057383/patch-decorator-is-not-compatible-with-pytest-fixture

### how mock.patch works

unittest patch is implemented using ExitStack. See its `__enter__` function, in
which, it uses the MagicMock/AsyncMock object to replay `self.attribute` of
`self.target`. See below short snippet.

```python
    if spec is None and _is_async_obj(original):
        Klass = AsyncMock
    else:
        Klass = MagicMock
    ...
    new = Klass(**_kwargs)
...
new_attr = new
...
try:
    setattr(self.target, self.attribute, new_attr)
```

So basically, it means if you have `@patch("tao.tao.read_object_new_format")`,
then it will call `setattr(tao.tao, 'read_object_new_format', new_attr)`.

### how to return multiple values in unittest mock

Below is function that implements `Mock.__call__`, so you can see that
`side_effect` is the first citizen and it can be either an exception or an
iterator.

```python
    def _execute_mock_call(self, /, *args, **kwargs):
        # separate from _increment_mock_call so that awaited functions are
        # executed separately from their call, also AsyncMock overrides this method

        effect = self.side_effect
        if effect is not None:
            if _is_exception(effect):
                raise effect
            elif not _callable(effect):
                result = next(effect)
                if _is_exception(result):
                    raise result
            else:
                result = effect(*args, **kwargs)

            if result is not DEFAULT:
                return result

        if self._mock_return_value is not DEFAULT:
            return self.return_value

        if self._mock_wraps is not None:
            return self._mock_wraps(*args, **kwargs)

        return self.return_value
```

Below is the `side_effect` setter. value is converted to iterator if is not
exception.

```python
def __set_side_effect(self, value):
    value = _try_iter(value)
    delegated = self._mock_delegate
    if delegated is None:
        self._mock_side_effect = value
    else:
        delegated.side_effect = value

def _try_iter(obj):
    if obj is None:
        return obj
    if _is_exception(obj):
        return obj
    if _callable(obj):
        return obj
    try:
        return iter(obj)
    except TypeError:
        # XXXX backwards compatibility
        # but this will blow up on first call - so maybe we should fail early?
        return obj
```

## pytest-cov

It depends on `coverage.py`.

## Pytest plugins

In order to understand how pytest plugin works, you need to read
[pluggy](https://github.com/pytest-dev/pluggy) first. Pytest plugin system is
based on it. The `pluggy` flow is straightforward. You define a hook
specification, i.e., `hookspec`, then define several plugins, i.e., `hookimpl`
and register them under this `hookspec`. One hook spec is executed as 1:N
mapping. N here is the number of hooks registered under a single hook spec.
Also, the registration order matters. Plugins are executed in the reverse
order.

Note that the example on [pluggy](https://github.com/pytest-dev/pluggy) front
page writes

```python
hookimpl = pluggy.HookimplMarker("myproject")

class Plugin_1:
    """A hook implementation namespace."""

    @hookimpl
    def myhook(self, arg1, arg2):
        print("inside Plugin_1.myhook()")
        return arg1 + arg2
```

Here, hook implementation marker `@hookimpl` is not required to register method
`myhook`. If you read the code carefully, you will see that `@hookimpl` is only
used to specify hook implementation options. Without this annotation, the
registered function has default options. Read `parse_hookimpl_opts` if you are
interested.

As said in the [basic flow](#basic-flow) section, during startup, Pytest
creates a `Config` object that has an plugin manager. Let's talk about plugin
manager constructor first. In the constructor, plugin manager add all hook
specs as below.

```
self.add_hookspecs(_pytest.hookspec)
```

You can take a look at file `src/_pytest/hookspec.py` to see all available
hooks inside Pytest. You are probably already familiar with
`pytest_collection_modifyitems`, `pytest_pyfunc_call` and etc. After adding
hook specs, plugin manger registers itself as a plugin.

```
self.register(self)
```

Therefore, Pytest plugin manager is both a plugin manager and a plugin.

During the initialization process, packages passed as CLI args `-p xx` will be
registered and a set of default packages will be registered too. See below.

```python
essential_plugins = (
    "mark",
    "main",
    "runner",
    "fixtures",
    "helpconfig",  # Provides -p.
)

default_plugins = essential_plugins + (
    "python",
    "terminal",
    "debugging",
    "unittest",
    "capture",
    "skipping",
    "legacypath",
    "tmpdir",
    "monkeypatch",
    "recwarn",
    "pastebin",
    "nose",
    "assertion",
    "junitxml",
    "doctest",
    "cacheprovider",
    "freeze_support",
    "setuponly",
    "setupplan",
    "stepwise",
    "warnings",
    "logging",
    "reports",
    "python_path",
    *(["unraisableexception", "threadexception"] if sys.version_info >= (3, 8) else []),
    "faulthandler",
)

```

Before going to the main execution loop. Let's talk about 3rd-party plugins.
Pytest automatically load plugins with `entry_points = pytest11`. Checkout
https://setuptools.pypa.io/en/latest/userguide/entry_point.html#entry-points-for-plugins
for `entry points` concept in python setuptools. When does Pytest register all
these 3rd-party plugins? It is also at the initialization stage:

```
_pytest/config/__init__.py:main -> _prepareconfig ->
    -> pluginmanager.hook.pytest_cmdline_parse -> parse -> _preparse
      -> self.pluginmanager.load_setuptools_entrypoints("pytest11")
```

Now, let's talk about the main execution flow of Pytest. As said above, Pytest
will register a lots of hook specs in the initialization stage. Among these
hook specs, below one is special.

```
@hookspec(firstresult=True)
def pytest_cmdline_main(config: "Config") -> Optional[Union["ExitCode", int]]:
    """Called for performing the main command line action. The default
    implementation will invoke the configure hooks and runtest_mainloop.

    Stops at first non-None result, see :ref:`firstresult`.

    :param pytest.Config config: The pytest config object.
    """
```

Because this hook is the main body of Pytest main function. See
[basic flow](#basic-flow) section above. As you can see, this hook is a
`firstresult` hook, namely, it stops at first non-None result. If you want to
write a plugin and provide this hook, you muster either return None or throw
exception. Folder `tutorials/pytest-plugin-deep-dive` has an example that shows
what plugins are loaded for the example test. You can see the first is
`_pytest/main.py` and the last is `pytest_split/plugin.py`. As said before,
plugins run in reverse order, so `pytest-split` is the first to run. Checkout
[pytest-split](https://github.com/jerry-git/pytest-split/blob/40fcefbea693b35360f23ac4c420bfeb173abdd0/src/pytest_split/plugin.py#L80).
pytest-split does some argument validation and return None. `_pytest/main.py`'s
code is below

```python
def pytest_cmdline_main(config: Config) -> Union[int, ExitCode]:
    return wrap_session(config, _main)


def _main(config: Config, session: "Session") -> Optional[Union[int, ExitCode]]:
    """Default command line protocol for initialization, session,
    running tests and reporting."""
    config.hook.pytest_collection(session=session)
    config.hook.pytest_runtestloop(session=session)

    if session.testsfailed:
        return ExitCode.TESTS_FAILED
    elif session.testscollected == 0:
        return ExitCode.NO_TESTS_COLLECTED
    return None
```

You can see that Pytest basically has two steps: `pytest_collection` and
`pytest_runtestloop`.

`pytest_collection` is the process to collect all tests in the codebase or
passed from command line. The core concept in this process is `Collector`.
`_pytest/python.py` has a few commonly used collectors: `Class`, `Module` and
`Package`. For example, when I run `pytest -s test2.py`. Function
`Session:perform_collect` returns a `test2.py` `Module` collector. Then, we
call `Moule:collect()` to collect all test functions in it. It is kind of
traversing a tree structure.

TODO: read `pytest_runtestloop`.

So far, I can see that the Pytest plugin system/logic makes whole sense and is
smartly designed.

Below I will discuss a few commonly used hooks.

#### pytest_addoption

This is a historic hook meaning that once new plugin is registered and this
plugin has defined this hook implementation, then this hookimpl will be called
using previous called parameters. The first time this hook is called is in
`Config` initialization.

```
self.hook.pytest_addoption.call_historic(
    kwargs=dict(parser=self._parser, pluginmanager=self.pluginmanager)
)
```

#### pytest_pycollect_makeitem

Inside python collector `Module` or `Class`, the `collect` function will
iterate the dictionary of this module/class to find all test functions. See
blow.

```
  def collect(self) -> Iterable[Union[nodes.Item, nodes.Collector]]:
      if not getattr(self.obj, "__test__", True):
          return []

      # Avoid random getattrs and peek in the __dict__ instead.
      dicts = [getattr(self.obj, "__dict__", {})]
      if isinstance(self.obj, type):
          for basecls in self.obj.__mro__:
              dicts.append(basecls.__dict__)

      # In each class, nodes should be definition ordered.
      # __dict__ is definition ordered.
      seen: Set[str] = set()
      dict_values: List[List[Union[nodes.Item, nodes.Collector]]] = []
      ihook = self.ihook
      for dic in dicts:
          values: List[Union[nodes.Item, nodes.Collector]] = []
          # Note: seems like the dict can change during iteration -
          # be careful not to remove the list() without consideration.
          for name, obj in list(dic.items()):
              if name in IGNORED_ATTRIBUTES:
                  continue
              if name in seen:
                  continue
              seen.add(name)
              res = ihook.pytest_pycollect_makeitem(
                  collector=self, name=name, obj=obj
              )
              if res is None:
                  continue
              elif isinstance(res, list):
                  values.extend(res)
              else:
                  values.append(res)
          dict_values.append(values)

      # Between classes in the class hierarchy, reverse-MRO order -- nodes
      # inherited from base classes should come before subclasses.
      result = []
      for values in reversed(dict_values):
          result.extend(values)
      return result
```

A good example of this hook is `pytest-asyncio` library, which has this hook
that adds async functions to the result set.

### How to debug plugins

I recently encountered a problem that I cannot reproduce `pytest-split` groups
locally compared to CI. It turns out the issue is simple. That is the pytest
rootdir is wrong. See the
[document](https://docs.pytest.org/en/6.2.x/customize.html#initialization-determining-rootdir-and-configfile)
and relevant code
[here](https://github.com/pytest-dev/pytest/blob/04be900d0677791d97e955b42440627b1818fbcb/src/_pytest/config/findpaths.py#L191)
When using `pyest-split` we need to pass a parameter
`--durations-path ~/Downloads/.test_durations`. This parameter is mistakenly
used by pytest to determine the rootdir.

How I found this issue? I put blow plugin definition in `conftest.py`.

```python
@pytest.hookimpl(tryfirst=True)
def pytest_collection_modifyitems(
    session, config, items
) -> None:
    import json
    with open(config.option.durations_path) as f:
        cached_durations = json.loads(f.read())
    splits: int = config.option.splits
    group_idx: int = config.option.group
    from pytest_split import algorithms
    algo = algorithms.Algorithms[config.option.splitting_algorithm].value
    breakpoint()
    groups = algo(splits, items, cached_durations)
    group = groups[group_idx - 1]
    # import IPython; IPython.embed()
    return None
```
