---
layout: post
title: Chrome -- Getting Started
date: 2025-05-21 20:53 -0700
categories: [frontend]
tags: [frontend, chrome]
---

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
