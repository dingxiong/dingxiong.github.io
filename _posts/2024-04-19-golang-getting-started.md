---
layout: post
title: Golang -- Getting started
date: 2024-04-19 19:34 -0700
categories: [programming-language, golang]
tags: [golang]
---

## Build

Follow [this guild](https://go.dev/doc/install/source) to build golang from
source.

```
cd src && ./all.bash

# If you do not want to run tests, then
./make.bash
```

Also, remember to clear cache `go clean --cache` after rebuilding go, or use
`-a` argument when running `go build`.

## Go tools

- `go fmt ./...` formats go code.
- `golangci-lint` is the most used linter for Golang.
- [deadcode detector](https://go.dev/blog/deadcode)

## Debug

`go mod edit -replace` is the way to go to use a customized branch or commit of
a dependence.

### delve

`dlv attach pid` is a useful command to see the internals of a running process.
Under the hood, delve calls Unix system call `ptrace`. Inside Alpine

```
apk add --update --no-cache go
go install github.com/go-delve/delve/cmd/dlv@latest
export PATH=$PATH:/root/go/bin
```

Not sure why using alpine, delve cannot find the debug symbols.
