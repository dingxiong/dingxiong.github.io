---
layout: post
title: Linux -- Getting started
date: 2024-03-09 23:07 -0800
categories: [os, linux]
tags: [linux, memory]
---

It is so hard to compile Linux kernel on my Macbook M1. Since I only need the
compilation database, I decided to build it in a Linux box, and then copy the
the file `compile_commands.json` out.

```bash
cd ~/code/linux

# install dependencies
sudo apt-get install flex libelf-dev bc bear liblz4-tool

# Disable certificates
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS

# enable cgroup
scripts/config --enable CONFIG_CGROUPS
scripts/config --enable CONFIG_RESOURCE_COUNTERS
scripts/config --enable CONFIG_MEMCG
scripts/config --enable CONFIG_MEMCG_SWAP
scripts/config --enable CONFIG_MEMCG_KMEM

# unselect all options
make menuconfig

# build
bear -- make -j4

# copy it out
scp ...

# fix the commands
sed -i 's;home/admin;Users/xiongding;g' compile_commands.json
```

It only took a few minutes because I disabled all options at the menu config
step. Now, `clangd` jumps perfectly!

To avoid compilation again in the future, I uploaded the compilation database
file to
[gist](https://gist.github.com/dingxiong/c4c02343cf68382f0fc83b505c61443d).
