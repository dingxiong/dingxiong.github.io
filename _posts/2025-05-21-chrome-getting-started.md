---
layout: post
title: Chrome -- Getting Started
date: 2025-05-21 20:53 -0700
categories: [frontend]
tags: [frontend, chrome]
---

## Build

Follow the
[official doc](https://chromium.googlesource.com/chromium/src/+/main/docs/mac_build_instructions.md).

```
cd code
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git --depth=1
export PATH="$HOME/code/depot_tools:$PATH"

mkdir chromium && cd chromium
caffeinate fetch --no-history chromium
cd src
gn gen out/default

# check `ulimit -n`. If too small, need increase it.
ulimit -n 65536
caffeinate autoninja -C out/default chrome
python tools/clang/scripts/generate_compdb.py -p out/default/ -o ./compile_commands.json

# run it
$HOME/code/chromium/src/out/default/Chromium.app/Contents/MacOS/Chromium
```

It took about 15 min to checkout and 6 hours to build on my MacBook M1. This
repo ships with llvm and it builds llvm on the way!

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

The corresponding code is
[here](https://github.com/chromium/chromium/blob/a10195ea0eb340a429b3f21178853db210e17579/third_party/blink/renderer/bindings/core/v8/local_window_proxy.cc#L302).
This global proxy maps all accesses to the internal proxied global object, and
then this global object in JS is hooked with the corresponding C++ object
through V8. Why do we need a proxy but not expose the global object directly?
Please see comment
[here](https://github.com/chromium/chromium/blob/a10195ea0eb340a429b3f21178853db210e17579/third_party/blink/renderer/bindings/core/v8/window_proxy.h#L47).

When you type `window` in the dev tools console, JS tries to locate this
attribute in the current scope. If not found, it goes above to global scope and
looks for the same attribute on the global proxy. Therefore, code `window`
really means `<the-global-proxy>.window`. This attributes exits and it is
clearly specified inside the
[IDL file](https://github.com/chromium/chromium/blob/a10195ea0eb340a429b3f21178853db210e17579/third_party/blink/renderer/core/frame/window.idl#L39).
The corresponding implementation is
[here](https://github.com/chromium/chromium/blob/a10195ea0eb340a429b3f21178853db210e17579/third_party/blink/renderer/core/frame/dom_window.cc#L314).
It returns itself!

`self` is defined in the IDL file and the implementation is the same as
`window`, so `self` is a simple alias for `window`.

`window` or `self` cannot be used in the browser environment. In node, the
global object is called `global`. To consolidate these names, ECMAScript 2020
propose a name `globalThis`.

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

## Cookie

HttpOnly cookie

Session cookie vs permanent cookie
