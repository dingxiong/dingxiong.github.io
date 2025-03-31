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

## Node

### nvm

`nvm` is a pure bash implementation of node version management. It installed a
shell function
[nvm](https://github.com/nvm-sh/nvm/blob/ffec9fec724da725013d5b50e763908113983fc3/nvm.sh#L3000)
for you. When you starts the shell, it injects
`$HOME/.nvm/versions/node/<version>/bin` to the $PATH environment variable, so
you can happily use node command.

How does it switch node version? When you run `nvm use <version>`. It simply
regex replaces the version in $PATH. See
[code](https://github.com/nvm-sh/nvm/blob/ffec9fec724da725013d5b50e763908113983fc3/nvm.sh#L981).

We use `nvm install <version>` to add a new version of node under directory
`~/.nvm/versions`. By default, only `npm` package is installed. You can verify
it by `npm ls -g`. By the way, as a newbie to the JS world. I was curious where
these `npm` global packages are installed. `npm config get prefix` returns the
`~/.nvm/versions/node/<version>`. I still haven't figured out how `nvm` sets
this prefix correctly.
