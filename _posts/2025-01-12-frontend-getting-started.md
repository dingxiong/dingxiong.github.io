---
layout: post
title: Frontend -- Getting Started
date: 2025-01-12 12:27 -0800
categories: [frontend]
tags: [frontend]
---

## Style and component libraries

- blueprintjs
- [ant design](https://ant.design/)
- [Material-UI](https://mui.com/)  
  google style UI. Better for mobile. Not recommended for web.

- Bootstrap
- tailwind  
  This
  [blog post](https://adamwathan.me/css-utility-classes-and-separation-of-concerns/)
  from the author is good to read. It explains the two directions of hooking
  html and css. In the first glance, tailwind looks like inline styles, but in
  tailwind, you can't just pick any value want; you have to choose from a
  curated list.

## Webpack

both `babble` and `ts-loader` can compile typescript to javascript.

### Source map

Webpack will do a sequence of transformations of the source code to get a
minified code. In order to trace back to the original file/line when debugging,
we need source map. See [this post](https://www.bugsnag.com/blog/source-maps)
for details.

## Yarn

yarn 2 is called `Berry`. When init a yarn library, do `yarn init -2`.
Otherwise, it will use yarn 1.

Sometimes, I got warning like below when installing pacakges

```
warning "@blueprintjs/datetime > react-day-picker@7.4.9" has incorrect peer dependency "react@~0.13.x || ~0.14.x || ^15.0.0 || ^16.0.0 || ^17.0.0"
```

See
[this article](https://blog.bitsrc.io/understanding-peer-dependencies-in-javascript-dbdb4ab5a7be)
for the peer dependency concept. Meanwhile, you can use
`yarn install --check-files` to check the issue.

## Next.js

## SWC: speedy web compiler

written in Rust.
