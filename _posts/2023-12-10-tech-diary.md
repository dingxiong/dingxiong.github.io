---
title: Tech diary
date: 2023-12-10 23:39 -0800
categories: [diary]
tags: [diary]
---

Some random thoughts or learnings.

## 2023-12-24 Sun

I read a few videos from Prof. Andy. I just feel that the two most difficult
problems in RDMS are query optimization and concurrency control.

## 2023-12-16 Sat

I need code folding in my blog, and this post
https://github.com/cotes2020/jekyll-theme-chirpy/issues/436 perfectly solved my
problem.

## 2023-12-15 Fri

This is really a nice series about Mysql Internals
https://www.zhihu.com/column/c_1453135878944567296

## 2023-12-10 Sun

When I was watching Prof. Andy's CMU class talk
[Query Execution 1](https://www.youtube.com/watch?v=I0QdCSu06_o&list=PLSE8ODhjZXjaKScG3l0nuOiDTTqpfnWFf&index=12),
one part stroke me

> Expression trees are flexible but slow. JIT compilation can (sometimes) speed
> them up.

This is very interesting. This reminds me of the days when we wrote the
execution engine for the matching service DSL. I should definitely spend some
time reading Postgres JIT.
