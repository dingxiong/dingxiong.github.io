---
layout: post
title: Python AWS
date: 2024-03-04 17:28 -0800
categories: [programming-language, python]
tags: [python, aws]
---

`botocore` and `aiobotocore` are the two fundamental AWS Python SDK. Most
people probably only interact with the higher level abstraction packages such
as `boto3` and `aioboto3`.

## How are different services clients implemented?

AWS has so many service, so its engineers decided to generate service clients
automatically. The code
[class ClientCreator](https://github.com/boto/botocore/blob/b9cd50770a279147d26ddbad8be48c67802d5bdb/botocore/client.py#L541).
That's why the source code of all methods are the same. See below example.

```python
In [91]: s3 = session.client("s3")
In [92]: x = await s3.__aenter__()
In [97]: inspect.getsource(x.head_object)
Out[97]: '        def _api_call(self, *args, **kwargs):\n            # We\'re accepting *args so that we can give a more helpful\n            # error message than TypeError: _api_call takes exactly\n            # 1 argument.\n            if args:\n                raise TypeError(\n                    f"{py_operation_name}() only accepts keyword arguments."\n                )\n            # The "self" in this scope is referring to the BaseClient.\n            return self._make_api_call(operation_name, kwargs)\n'
```

Async methods are no different. The
[AioBaseClient](https://github.com/aio-libs/aiobotocore/blob/e8a3b8e03dbf010ad2cddd751dfdf759a7df0780/aiobotocore/client.py#L324C15-L324C29)
overwrites function `_make_api_call`: it overwrites a sync function in base
class with an async function in the derived class. Therefore, all generated
methods become async. Blow snippet shows how it works.

```python
In [94]: async def f_dervied():
    ...:     return 10

In [95]: def bar():
    ...:     return f_dervied()

In [96]: await bar()
Out[96]: 10
```

This "clever" design makes some analysis difficult: in the above example,
though `bar()` can be awaited, but `bar` itself is not a coroutine function.
See blow.

```python
In [71]: s3 = session.client("s3")
In [72]: c = await s3.__aenter__()
In [75]: inspect.isasyncgenfunction(c.head_object)
Out[75]: False
```

It is not the end of weirdness! For some other functions, the inspection shows
"expected" result.

```python
In [98]: inspect.iscoroutinefunction(x.generate_presigned_url)
Out[98]: True
```

It turns out that `generate_presigned_url` method is explicitly
defined/overwritten. See
[code](https://github.com/aio-libs/aiobotocore/blob/e8a3b8e03dbf010ad2cddd751dfdf759a7df0780/aiobotocore/signers.py#L172).
