---
layout: post
title: Python -- Web
date: 2024-03-05 14:40 -0800
categories: [programming-language, python]
tags: [python]
---

## What is WSGI?

Web Server Gateway Interface. Work flow blow

```
client requests -> web server/gateway -> WSGI server -> Python web application/framework.
```

- Web server/gateway: NGINX, Apache, etc
- WSGI server: Gunicorn, uWSGI, etc
- Python web application/framework: Django, Flask, etc

## WSGI server

### Gunicorn

Gnuicorn uses multiple precess/thread model. The core code about process/thread
management is in `arbiter.py`. One interesting thing about Gunicorn is that is
uses `ast` to parse the input WSGI application entry method.

Checkout more about Gunicorn [here](gunicorn.md).

### Why put Nginx before Gunicorn?

For slow clients cases. See
https://serverfault.com/questions/220046/why-is-setting-nginx-as-a-reverse-proxy-a-good-idea

### gevent vs eventlet

Gunicorn can use coroutine worker class gevent or eventlet Both are built on
top of `greenlet`, Greenlet is a kind of cooperative thread, which are logical
threads that share a single physical thread, and use IO polling for cooperative
execution. It is every similar to asynio in python. It is similar to Java
thread too, but Java thread is time slice based.

### eventlet

- GreenThread: a subclass of `greenlet`, which has `switch` function that
  preempts another `GreenThread`.
- hub: maintains a queue of GreenThreads, and uses `epoll` or `kqueue` to
  register IO sockets such that it can orchestrate these `GreenThreads`. It is
  the main object that realize cooperative coroutine. Note, `hub` is a thread
  local object. Here thread means physical thread.
- GreenPool: A fixed size of collection of `GreenThread`s. It provides function
  `spawn` to create a new `GreenThread`. Also, it uses semaphore to maintain
  the pool size.

## What is CGI?

TODO

## Gunicorn

Gunicorn is a Python web server gateway interface (WSGI) HTTP server that helps
Python web applications run faster and more securely. It supports multiple
worker processes, load balancing, graceful restarts, and error handling.
Gunicorn is widely used and popular among Python developers due to its ease of
use and scalability.

## Internal implementations

### Arbiter

Arbiter is gunicorn's worker controller. It is a good material to learn linux
programming. It spawns worker process by using `fork` system call. It manages
child process by using SIGCHLD signal handler and `waitpid` system calls. It
use PIPE to do IPC (interprocess communication). Also, it handler systemd as
well, so it can be run inside systemd.

## Flask

### Application context

`flask.g` is a global variable that is a proxy to the current application
context. It is interesting how it is implemented as push/pop operation when
context enters and exits, probably similar to `ddog` trace implementation.

TODO: study the proxy implementation of flask.g

### Sessions

Flask has builtin client-side session support meaning that session data is
stored in browser cookies. The benefit of client-side session is that you do
not need to worry about data consistency if you have multiple web servers when
storing session data on server side. However, as pointed out by
[this article](https://blog.miguelgrinberg.com/post/how-secure-is-the-flask-user-session)
we should not store sensitive data in client-side session.

On the other hand, most Flask apps use `flask-login` plugin. It uses flask
built-in session to manage login data. See `flask/globals.py`.

In this sense, server side session is necessary to store sensitive session
data. Plug in `flask-session` has some good stuff. It can be configured to use
different data stores like Redis or Memcached.

### Flask-login

Flask-login is a package that makes login session management easy. The core
flow is as following:

1. After user successful login, "user_id" will be
   [saved in session](https://github.com/maxcountryman/flask-login/blob/486c456907e307e363d0c334fb2c079d4fa25d55/flask_login/utils.py#L162)
   and
   [sent to client side as a cookie](https://github.com/pallets/flask/blob/f976d5bb88216e921a96998f767df31d7039e4ef/src/flask/sessions.py#L412).
2. When the user visits another page, it sends the cookie as a http header to
   the server. Flask-login defines a proxy variable
   [current_user](https://github.com/maxcountryman/flask-login/blob/486c456907e307e363d0c334fb2c079d4fa25d55/flask_login/utils.py#L26).
   If you trace down the loader behind, you will see that the loader first
   extracts `user_id` from the session and then calls a function annotated like
   below to load user object.
   ```
   @login_manager.user_loader
   def load_user(user_id):
      ...
   ```
   Flask-login assumes the user returned from this function is a sub type of
   [UserMixin](https://github.com/maxcountryman/flask-login/blob/486c456907e307e363d0c334fb2c079d4fa25d55/flask_login/mixins.py#L12),
   which explicitly has `is_authenticated = True`.
3. Flask-login provides a annotator `login_required` that checks if
   [current_user.is_authenticated == True](https://github.com/maxcountryman/flask-login/blob/486c456907e307e363d0c334fb2c079d4fa25d55/flask_login/utils.py#L259).

You can see that we actually store critical information in our browser's
cookie: `user_id`. Anyone who has the cookie can interact with the backend
server representing this user. Then do I need to worry about a malicious user
steals my cookie? Most time No! There are two mechanism to protect you. First,
flask has CSRF projection if enabled. I believe all production flask servers
have enabled it :). Second, your website is using https, so nobody can
intercept your traffic and decode out your cookie.

Let's see an example of how a session cookie looks like. Below is part of a
cookie header intercepted.

```
session=.eJwlz0FuAyEMBdC7sO5IGDCGXGZkG7uNmk6qIVlVvXupsvLm_6_nn7D7afMjXJxv097Cfh3hEtSok8MQz8Ito6emhD2rZHZprBVjhkZdING6XrF3iuYoQtKrJhrcOkNWTo7N0UHMvZZWtHEZaFpIuA4CVC6eEyoIWarszS0syLedX3zY8QiXx_lcNJ2n74_7px1LCDAK2QoP0OZcki0S9Qhmqq1Q54SUM6yl2_39euzT5rzej9d_GGOLYrz16HErY-DGUn1zHzZEJMH4Nzynna8CxFZrCb9_07RaaA.FqDA-Q.XVXv83QHvGngOAFaK2Dy1Kwa1RQ;
```

By default, flask use key `session` for its session cookie. Ignore the ending
";", that is a delimiter that separates different cookie keys. Flask uses a
package `itsdangerous` to
[sign cookies](https://github.com/pallets/flask/blob/f976d5bb88216e921a96998f767df31d7039e4ef/src/flask/sessions.py#L411).
If you read the internals of this package, you will find the basic flows is
like

1. base64url session and possibly zip it. Let's call the result
   `ZippedBase64edSession`.
2. base64url timestamp, denoted `Base64edTs`
3. use `ZippedBase64edSession` to generated a signature and base64 encode it,
   denoted `Base64edSignature`

Then the session string above is
`ZippedBase64edSession.Base64edTs.Base64edSignature`. Yes, simply concatenate
three components using period symbol. Note that only step #3 uses a secret key.
We can easily decode session content without knowing this secret. Let's do it!

```
In [9]: session_cookie=b".eJwlz0FuAyEMBdC7sO5IGDCGXGZkG7uNmk6qIVlVvXupsvLm_6_nn7D7afMjXJxv097Cfh3hEtSok8MQz8Ito6emhD2rZHZprBVjhkZdI
   ...: NG6XrF3iuYoQtKrJhrcOkNWTo7N0UHMvZZWtHEZaFpIuA4CVC6eEyoIWarszS0syLedX3zY8QiXx_lcNJ2n74_7px1LCDAK2QoP0OZcki0S9Qhmqq1Q54SUM6yl
   ...: 2_39euzT5rzej9d_GGOLYrz16HErY-DGUn1zHzZEJMH4Nzynna8CxFZrCb9_07RaaA.FqDA-Q.XVXv83QHvGngOAFaK2Dy1Kwa1RQ"
In [10]: session = session_cookie.rsplit(b".", 2)[0]
In [11]: from itsdangerous import URLSafeTimedSerializer
In [12]: s = URLSafeTimedSerializer('fake-secret')
In [13]: s.load_payload(session)
Out[13]:
{'_fresh': False,
 '_id': 'ce797f1dbf3ba835f28c7593cb3afb8ac65031879b127187f659970ef5bb7b96c27da89a13ca2f58f5f1beff6484c8a4d5ec47ba6d715ca4f325c1b7e26af8fe',
 '_permanent': True,
 'csrf_token': '11d47eaf8d1c8fa42e6507901eecc8479a257331',
 'login_session_id': '50080bea-90f0-4dd5-ab6f-ffdedbbb21de',
 'user_id': '108664'}
```

Amazing, right?

### plugins

- flask-login
- flask-session
- flask-security
  - Not sure how useful this plugin is. It just adds a few wrappers on top of
    other plugins.

### Testing

https://flask.palletsprojects.com/en/2.1.x/
