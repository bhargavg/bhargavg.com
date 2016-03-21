---
layout: post
title:  "Building swift toolchain on OS X and Ubuntu-15.10"
date:   2015-03-15 00:20:00
categories: Swift
---

> Tl;Dr: Use `utils/build-toolchain` script in `Swift` repo to build the toolchain on both OS X and Ubuntu

[API Design Guidelines Porposal](https://github.com/apple/swift-evolution/blob/master/proposals/0023-api-guidelines.md) introduced considerable breaking changes to Swift API.

Foundation, XCTest and Swift Package Manager have already merged PR's adopting these changes, meaning, we can no longer use the latest (Mar 1st, 2016) toolchain for developing/contributing to Swift.

As it turns out, creating toolchain for ourselves is not that difficult. There is a nifty little script `build-toolchain` under `utils` directory in `Swift` repository that can be used to build toolchains.

## Understanding build scripts

There are four important files that help in building Swift from sources:

1. [build-toolchain](https://github.com/apple/swift/blob/e1ad6d80ee4e5aeb77c6c39e221f9f10f41e8056/utils/build-toolchain) - This will initialize different parameters that will be passed to `build-script`
2. [build-presets.ini](https://github.com/apple/swift/blob/e1ad6d80ee4e5aeb77c6c39e221f9f10f41e8056/utils/build-presets.ini) - Defines multiple presets for building swift with different flags/settings. `buildbot_linux` and `buildbot_osx` preset will be used for building toolchains on Ubuntu and OS X respectively
3. [build-script](https://github.com/apple/swift/blob/e1ad6d80ee4e5aeb77c6c39e221f9f10f41e8056/utils/build-script) - Python script that acts as frontend for `build-script-impl`. This does initial parameter validation, adds platform dependent arguments etc.,
4. [build-script-impl](https://github.com/apple/swift/blob/e1ad6d80ee4e5aeb77c6c39e221f9f10f41e8056/utils/build-script-impl) - Workhorse that actually builds the Swift

## Prerequisites

On Ubuntu-15.10 64-bit server edition, following are dependencies that needs to be installed prior to running the build script:

```bash
$ sudo apt-get install git             \ 
                       binutils        \ 
                       build-essential \ 
                       icu-devtools    \
                       clang           \ 
                       cmake           \ 
                       gcc             \ 
                       libbsd-dev      \ 
                       libedit-dev     \ 
                       libncurses-dev  \ 
                       libsqlite3-dev  \
                       libtool         \ 
                       libxml2-dev     \ 
                       pkg-config      \ 
                       python          \ 
                       python-dev      \ 
                       sqlite3         \ 
                       uuid-dev        \ 
                       ninja-build
```

## Fetching the dependencies

Inorder to create Swift toolchain, we need to clone its dependencies along side with Swift in the same folder.

As seen [here](https://github.com/apple/swift/blob/e1ad6d80ee4e5aeb77c6c39e221f9f10f41e8056/utils/build-script-impl#L925), the `build-script-impl` expects the dependencies to have sepecific names

```bash
$ git clone --depth 1 https://github.com/ninja-build/ninja ninja
$ git clone --depth 1 https://github.com/apple/swift swift
$ git clone --depth 1 https://github.com/apple/llvm llvm
$ git clone --depth 1 https://github.com/apple/swift-cmark cmark
$ git clone --depth 1 https://github.com/apple/swift-lldb lldb
$ git clone --depth 1 https://github.com/apple/swift-llbuild llbuild
$ git clone --depth 1 https://github.com/apple/swift-package-manager swiftpm
$ git clone --depth 1 https://github.com/apple/swift-corelibs-xctest
$ git clone --depth 1 https://github.com/apple/swift-corelibs-foundation
$ git clone --depth 1 https://github.com/apple/swift-corelibs-libdispatch
```

## Building

```bash
$ cd swift
$ utils/build-toolchain local.swift
```

The last command will initiate the building process which last from 1hr to 3hr, depending on your system speed.

The toolchain will be generated with the name `swift-LOCAL-YYYY-MM-DD-a-osx.tar.gz` and can be extracted using 

```bash
tar -xvf swift-LOCAL-YYYY-MM-DD-a-osx.tar.gz
```

## Disabling Tests

Tests take huge amounts of time to run. I disabled tests by using the following simple steps:

1. After running `build-toolchain local.swift` command, the actual command that gets executed will be printed on console as 
```
utils/build-script: using preset 'buildbot_linux', which expands to 
---COMMAND-FOLLOWS-HERE---
```
2. Copy the command completely
3. Remove `--tests` flag and add `skip-test-osx`, `skip-test-ios`, `skip-test-tvos`, `skip-test-watchos` flags (depending on your platform)
4. If you still see tests running, that means your toolchain is ready, you can kill the tests by `Ctrl-C`


That's all folks...!
