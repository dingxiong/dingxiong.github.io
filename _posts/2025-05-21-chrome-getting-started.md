---
layout: post
title: Chrome -- Getting Started
date: 2025-05-21 20:53 -0700
categories: [frontend]
tags: [frontend, chrome]
---

## The global window object

I am always wondering about the usage of `window` variable inside JS code. It
seems a global variable, but where is it defined and what attributes it has?

First, let's clarify one thing. When a frame loads, Chrome initializes a JS
context, and automatically creates a global proxy inside this context. The
relationship is like below

```
[ LocalFrame ]
      |
      v
[ LocalDOMWindow ] ⇄ [ LocalWindowProxy ] ⇄ [ V8 JS Global Object (window) ]
```

This global proxy maps all access to the internal proxied global object, and
then this global object in JS is hooked with the corresponding C++ object
through V8. Why do we need a proxy but not expose the global object directly?
Please see comment
[here](https://github.com/chromium/chromium/blob/a10195ea0eb340a429b3f21178853db210e17579/third_party/blink/renderer/bindings/core/v8/window_proxy.h#L47).
This global proxy is used for interpreting JS code. When you type `window` in
the dev tools console, JS will try to locate this attribute in the current
scope. If not found, it goes above to global scope and looks for the same
attribute of the global proxy. Therefore, code line `window` really means
`<the-global-proxy>.window`. This attributes exits and it is clearly specified
inside the
[IDL file](https://github.com/chromium/chromium/blob/a10195ea0eb340a429b3f21178853db210e17579/third_party/blink/renderer/core/frame/window.idl#L39).
The corresponding implementation is
[here](https://github.com/chromium/chromium/blob/a10195ea0eb340a429b3f21178853db210e17579/third_party/blink/renderer/core/frame/dom_window.cc#L314).
It just return itself!

`self` is defined in the IDL file and the implementation is the same as
`window`, so `self` is just an alias for `window`. However, this is only valid
for browser, in node, the global object is called `global`. To consolidate
these names, ECMAScript 2020 propose a name `globalThis`.

## SCP (Content Security Policy)

The core logic is
[here](https://github.com/chromium/chromium/blob/a10195ea0eb340a429b3f21178853db210e17579/third_party/blink/renderer/core/frame/csp/source_list_directive.cc#L49).

1. If the directive contains _, it allows HTTP family protocols (http/https) or
   URLs matching the current page's scheme. This is why `script-src _` allows
   scripts from any HTTP/HTTPS source.
2. If directive contains 'self', check if URL matches the current origin.
3. Check against explicitly listed sources (like https://example.com)
4. Special handling for WebSocket URLs with wildcards.

## Blink

[How Blink works](https://docs.google.com/document/d/1aitSOucL0VHZa9Z2vbRJSyAIsAz24kX8LFByQ5xQnUg/edit?tab=t.0)
