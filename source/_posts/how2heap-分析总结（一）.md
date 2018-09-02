---
title: how2heap-分析总结（一）
abbrlink: 41139
date: 2018-08-06 23:33:01
tags:
password: 654321
---

# 前言

how2heap有了很大的变化，特在此学习，记录。

# 准备工作

由于glibc的变化，导致一些漏洞在某些版本无法利用，然而这个版本使用度依然很高，决定了这些漏洞还有利用价值。

glibc_build.sh

```shell
qianfa@qianfa:~/Desktop/how2heap$ cat glibc_build.sh 
#!/bin/bash

SRC="./glibc_src"
BUILD="./glibc_build"
VERSION="./glibc_versions"

if [[ $# < 2 ]]; then
    echo "Usage: $0 version #make-threads <-disable-tcache>"
    exit 1
fi

# Get glibc source
if [ -d "$SRC" ]; then
    cd $SRC
    git pull --all
else
    git clone git://sourceware.org/git/glibc.git "$SRC"
    cd "$SRC"
    git pull --all
fi

# Checkout release
git rev-parse --verify --quiet "release/$1/master"
if [[ $? != 0 ]]; then
    echo "Error: Glib version does not seem to exists"
    exit 1
fi

git checkout "release/$1/master"
cd -

# Build
if [ $# == 3 ] && [ "$3" = "-disable-tcache" ]; then
    TCACHE_OPT="--disable-experimental-malloc"
    SUFFIX="-no-tcache"
else
    TCACHE_OPT=""
    SUFFIX=""
fi

mkdir -p "$BUILD"
cd "$BUILD" && rm -rf ./*
../"$SRC"/configure --prefix=/usr "$TCACHE_OPT"
make -j "$2"
cd -

# Copy to version folder
mkdir -p "$VERSION"
cp "$BUILD/libc.so" "$VERSION/libc-$1$SUFFIX.so"
cp "$BUILD/elf/ld.so" "$VERSION/ld-$1$SUFFIX.so"
```

