---
layout: post
title: Chrome Preconnect Breaks Singly Threaded Servers
date: 2025-06-21 17:14 -0700
categories: [frontend]
tags: [frontend, chrome]
---

## Preconnection / Speculative Sockets

I encountered the same problem documented in
[this post](https://hackernoon.com/chrome-preconnect-breaks-singly-threaded-servers-95944be16400).
What I observed using tcpdump was that the first time I opened a page in
Chrome, two connections were established to my local Flask serer. One
connection, say `socket1`, finished the TCP three handshake and nothing else.
The other connection, say `socket2`, finished the handshake protocol and served
the request. The response was 307 and the target was a different origin, but
still served by this local server. Then I found it got stuck because Chrome
opened a 3rd connection, but `socket1` was not finished yet. The stupid
Flask/Werkzeug process was waiting for the http request on `socket1`. I
understand a redirect to a different origin but underneath the same server
confused Chrome, but still this speculative socket behavior wasted me so much
time to debug.

This post also claims that the problem can be solved with an Nginx reverse
proxy. I took a quick look at the source code, the relevant logic is
[here](https://github.com/nginx/nginx/blob/4eaecc5e8aa7bbaf9e58bf56560a8b1e67d0a8b7/src/http/modules/ngx_http_proxy_module.c#L1043).
Basically, Nginx starts to connect to the upstream, i.e., the backend service,
after the request body is fully received. You can also make the connection
happen a little bit earlier by disabling
[proxy_request_buffering](https://github.com/nginx/nginx/blob/4eaecc5e8aa7bbaf9e58bf56560a8b1e67d0a8b7/src/http/ngx_http_request_body.c#L208).
No matter what, Nginx connects to the upstream after the client side finishes
TCP handshake, sends the request line, http headers, and at least the firs
chuck of request body. So this is not a problem for Flask. Speculative sockets
stop at Nginx.

How does preconnection work? The relevant code is inside `loading_predictor.cc`
and `resource_prefetch_predictor.cc`.

### Autocomplete, Preconnect and Prefetch

These three functionalities are different. Autocomplete means auto completing
url when you type in the Chrome address bar. Preconnect means opening
connections in advance. Prefetch goes one step further. It sends the request to
the backend server before you really do it. In the source file
`loading_predictor.cc`, you will see two names: `prefetch_manager` and
`preconnect_manager`. Do not mistake them!

When I was testing the same local server endpoint multiple times in a straight
line, and then repoen Chrome, I found Chrome sent a request to my local server
without me opening the page! It predicted I would open it! To be honest, I was
totally shocked when I saw it.

The relevant data can be found at `chrome://predictors/`. It only shows
Autocomplete and Prefetch data, but not Preconnection data.

See the blow example. Because I opened amazon.com too often, Chrome will auto
completes me even I just type `a` in the address bar.
![chrome-autocomplete](/assets/images/chrome-autocomplete.png)

### How to turn off preconnect

There are few flags controlling the preconnection behavior. First,
[this code](https://github.com/chromium/chromium/blob/c265152bd6f1fb9d3e4ae6e89678d229d7a2e635/chrome/browser/predictors/loading_predictor.cc#L67)
says Chrome opens two connections to the server by default for a new origin.
This place adds the request to `prediction->requests`, then where are these
requests executed? It is
[here](https://github.com/chromium/chromium/blob/c265152bd6f1fb9d3e4ae6e89678d229d7a2e635/chrome/browser/predictors/loading_predictor.cc#L380)
and then
[here](https://github.com/chromium/chromium/blob/c265152bd6f1fb9d3e4ae6e89678d229d7a2e635/chrome/browser/predictors/preconnect_manager.cc#L117).
The implementation of `IsEnabled` says that preconnection is enabled iff:

1. There is Chrome profile.
2. The
   [preload page setting](https://github.com/chromium/chromium/blob/c265152bd6f1fb9d3e4ae6e89678d229d7a2e635/chrome/browser/preloading/preloading_prefs.h#L41)
   is `default` or `extended`.

This means cognito or guest profile disables preconnection. Alternatively, you
can disable preload page option explicitly. See the below screenshot.

![chrome-preload-page-setting](/assets/images/chrome-preload-page-setting.png)

A tiny hint for you when testing preconnect by yourself. Sometimes, you expect
Chrome to initiate two connections, but it does not. You probably need to go to
[chrome://net-internals/](chrome://net-internals) to flush the existing
connection pool.
![chrome-net-internals](/assets/images/chrome-net-internals.png)

Btw, the logic is gated by several feature flags, for instance
[LoadingPredictorUseLocalPredictions](https://github.com/chromium/chromium/blob/453369e51838580f82405175820b35eb25c5f0e2/chrome/browser/predictors/predictors_features.cc#L21).
This feature has matured for long time, so you won't be able to find it in
`chrome://flags/`. Therefore it takes the default value is `true`.
