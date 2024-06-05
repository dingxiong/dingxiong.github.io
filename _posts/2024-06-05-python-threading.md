---
layout: post
title: Python -- Threading
date: 2024-06-05 15:02 -0700
categories: [programming-language, python]
tags: [python]
---

I spent quite some time today debugging a seemingly trivial problem when
working with Python `ThreadPoolExecutor`. The code is as follows.

```python
with concurrent.futures.ThreadPoolExecutor(max_workers=8) as executor:
    fs = [executor.submit(_download_s3_file, file_name) for file_name in gen_file_names()]
    done, _not_done = concurrent.futures.wait(fs, return_when=concurrent.futures.FIRST_EXCEPTION)
    for f in done:
        f.result()
```

What I tried to achieve is stopping the thread pool immediately when the first
exception shows up. I thought I am clever because `f.result()`
[throws](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Lib/concurrent/futures/_base.py#L401)
if the future has exception, so the program terminates early. However, what I
observed was that the thread pool continued running. More and more s3 files
were downloaded even if some of the jobs failed. The program seemed to have get
stuck at `f.result()`.

After debugging for some time, I realized that `f.result()` is not the problem.
It indeed threw an exception, but then it runs `ThreadPoolExecutor.__exit__`,
which calls `self.shutdown(wait=True)`. It was waiting for all threads to join
and this part got stuck!

OK. I came up with a quick fix.

```python
with concurrent.futures.ThreadPoolExecutor(max_workers=8) as executor:
    fs = [executor.submit(_download_s3_file, file_name) for file_name in gen_file_names()]
    done, _not_done = concurrent.futures.wait(fs, return_when=concurrent.futures.FIRST_EXCEPTION)
    for f in done:
        if f.exception():
            executor.shutdown(wait=False, cancel_futures=True)
            raise f.exception()
```

Parameter `cancel_futures=True` is critical. If not set, it gets stuck too.
This looks like an acceptable solution. But it is quite verbose. I do not think
I am the only one who wants to achieve this behavior, so there should be a more
elegant approach?

Finally, I came cross function
[map](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Lib/concurrent/futures/_base.py#L583).
It cancels all futures if the first exception is threw. So the updated version
is

```python
with concurrent.futures.ThreadPoolExecutor(max_workers=8) as executor:
    list(executor.map(_download_s3_file, gen_file_names()))
```

It is much better. Right?
