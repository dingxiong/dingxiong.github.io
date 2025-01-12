---
layout: post
title: AWS -- Lambda
date: 2025-01-12 12:37 -0800
categories: [aws]
tags: [aws, lambda]
---

I am setting up some webhooks using AWS Lambda these days. As you know Lambda
functions can be deployed using a docker image. AWS provides a set of
[base images](https://github.com/aws/aws-lambda-base-images) for various
languages.

I am curious how these base images work underneath.

## Python base image

The first log I saw when running a python Lambda image is

```
25 Jul 2023 05:20:01,256 [INFO] (rapid) exec '/var/runtime/bootstrap' (cwd=/asset, handler=)
```

So I went to the docker container and below is the content of bootstrap script.

```
bash-4.2# cat /var/runtime/bootstrap
#!/bin/bash
# Copyright 2018 Amazon.com, Inc. or its affiliates. All Rights Reserved.

export AWS_EXECUTION_ENV=AWS_Lambda_python3.10

if [ -z "$AWS_LAMBDA_EXEC_WRAPPER" ]; then
  exec /var/lang/bin/python3.10 /var/runtime/bootstrap.py
else
  wrapper="$AWS_LAMBDA_EXEC_WRAPPER"
  if [ ! -f "$wrapper" ]; then
    echo "$wrapper: does not exist"
    exit 127
  fi
  if [ ! -x "$wrapper" ]; then
    echo "$wrapper: is not an executable"
    exit 126
  fi
    exec -- "$wrapper" /var/lang/bin/python3.10 /var/runtime/bootstrap.py
fi
```

Basically, it simply runs `bootstrap.py` in the same folder

```
bash-4.2# cat bootstrap.py
"""
Copyright 2018 Amazon.com, Inc. or its affiliates. All Rights Reserved.
"""

import json
import logging
import os
import site
import sys
import time
import traceback
import warnings
import awslambdaric.__main__ as awslambdaricmain


def is_pythonpath_set():
    return "PYTHONPATH" in os.environ


def get_opt_site_packages_directory():
    return '/opt/python/lib/python{}.{}/site-packages'.format(sys.version_info.major, sys.version_info.minor)


def get_opt_python_directory():
    return '/opt/python'


# set default sys.path for discoverability
# precedence: LAMBDA_TASK_ROOT -> /opt/python/lib/pythonN.N/site-packages -> /opt/python
def set_default_sys_path():
    if not is_pythonpath_set():
        sys.path.insert(0, get_opt_python_directory())
        sys.path.insert(0, get_opt_site_packages_directory())
#     'LAMBDA_TASK_ROOT' is function author's working directory
#     we add it first in order to mimic the default behavior of populating sys.path and make modules under 'LAMBDA_TASK_ROOT'
#     discoverable - https://docs.python.org/3/library/sys.html#sys.path
    sys.path.insert(0, os.environ['LAMBDA_TASK_ROOT'])


def add_default_site_directories():
#     Set 'LAMBDA_TASK_ROOT as site directory so that we are able to load all customer .pth files
    site.addsitedir(os.environ["LAMBDA_TASK_ROOT"])
    if not is_pythonpath_set():
        site.addsitedir(get_opt_site_packages_directory())
        site.addsitedir(get_opt_python_directory())

def set_default_pythonpath():
    if not is_pythonpath_set():
#         keep consistent with documentation: https://docs.aws.amazon.com/lambda/latest/dg/lambda-environment-variables.html
        os.environ["PYTHONPATH"] = os.environ["LAMBDA_RUNTIME_DIR"]


def main():
    set_default_sys_path()
    add_default_site_directories()
    set_default_pythonpath()
    awslambdaricmain.main([os.environ["LAMBDA_TASK_ROOT"], os.environ["_HANDLER"]])

if __name__ == '__main__':
    main()
```

OK. So it add a bunch of paths to `sys.path` including `LAMBDA_TASK_ROOT`. One
question confused me a lot is how to install 3rd party packages inside Lambda
docker image. Different posts gave different instructions. Some said all
packages should be installed inside `/assert` folder. Even AWS documents are
inconsistent among versions. So I did a quick experiment to figure out the
correct behavior. I installed random package using `pip install lz4`.

```
bash-4.2# pip show lz4
...
Location: /var/lang/lib/python3.10/site-packages
```

No surprise, it is installed under the standard python3.10 folder. Now check
`sys.path`.

```
bash-4.2# which python
/var/lang/bin/python

bash-4.2# /var/lang/bin/python
Python 3.10.12 (main, Jul 11 2023, 13:56:08) [GCC 7.3.1 20180712 (Red Hat 7.3.1-15)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys
>>> sys.path
['', '/var/lang/lib/python310.zip', '/var/lang/lib/python3.10', '/var/lang/lib/python3.10/lib-dynload', '/var/lang/lib/python3.10/site-packages']
```

OK. No hack! Just do `RUN pip install requirements.txt` in the Dockerfile!

Another thing to note is the `main()` function above. Its implementation is
[here](https://github.com/aws/aws-lambda-python-runtime-interface-client/blob/main/awslambdaric/__main__.py).
Basically, it is an endless while loop that handle incoming requests.
