---
layout: post
title: How SwiftPM Parses Manifest File
date: 2016-06-11 09:05
categories: Swift
disqus: true
uuid: 99F5E9A9-B0D8-4B61-A454-FFEF8BC82773
---

> Disclaimer: This post is outdated. Even though SwiftPM changed a lot since this post was written, the underlying idea is the same. So, please take with a pinch of salt.

SwiftPM uses `Package.swift` as a manifest file to capture information about the current package like, name of the package, its dependencies, targets, etc.,

Because `Package.swift` is written in Swift language, parsing it is not as straight forward as parsing `JSON` or `XML`. So, how does SwiftPM binary gets the package information?

Hints to this question lies in `Package.swift` of SwiftPM itself. Apart from defining root package and its dependencies, SwiftPM also specifies an explicit dynamic library product of PackageDescription module.

{% highlight swift %}
let dylib = Product(name: "PackageDescription",
                    type: .Library(.Dynamic),
                    modules: "PackageDescription")

products.append(dylib)
{% endhighlight %}

Why SwiftPM requires dynamic library for only this module?

## PackageDescription module

`PackageDescription` is a special module of SwiftPM for 2 reasons:

1. It is the only module that must be imported in every `Package.swift`.
2. It has no dependencies on other modules. It’s a design decision. This module is created to be used independently.

The main objective of this `PackageDescription` module is to

1. Provide APIs for creating Package.swift
2. Transmute the Package.swift from its Swifty form to a more consumable format.

Let’s see it in action.

## TOML

When swift build is invoked with verbose option, it will list out the following command:

{% highlight bash %}
PATH/TO/SNAPSHOT/usr/bin/swiftc \
    --driver-mode=swift \
    -I PATH/TO/SNAPSHOT/usr/lib/swift/pm \
    -L PATH/TO/SNAPSHOT/usr/lib/swift/pm \
    -lPackageDescription \
    PATH/TO/Package.swift \
    -fileno 3
{% endhighlight %}

So, the above command:

1. Compiles the `Package.swift`
2. Link it will `PackageDescription` dynamic library
3. Execute the resulting binary with `-fileno 3` argument

The `-fileno` argument takes a file descriptor. We can verify this by passing `1`, which is the file descriptor for `stdout`.

{% highlight bash %}
$ mkdir MySwiftyApp
$ cd MySwiftyApp
$ swift package init
Creating library package: MySwiftyApp
Creating Package.swift
Creating .gitignore
Creating Sources/
Creating Sources/MySwiftyApp.swift
Creating Tests/
Creating Tests/LinuxMain.swift
Creating Tests/MySwiftyApp/
Creating Tests/MySwiftyApp/MySwiftyAppTests.swift
$ PATH/TO/SNAPSHOT/usr/bin/swiftc \
    --driver-mode=swift \
    -I PATH/TO/SNAPSHOT/usr/lib/swift/pm \
    -L PATH/TO/SNAPSHOT/usr/lib/swift/pm \
    -lPackageDescription \
    PATH/TO/Package.swift \
    -fileno 1
[package]
name = "MySwiftyApp"
dependencies = []
testDependencies = []

exclude = []
{% endhighlight %}

Sweet…

The entire `Package.swift` is dumped as `TOML` representation on console !!!

## Implementation
Let’s look at how this is implemented.

All magic happens in `Sources/PackageDescription/Package.swift`. It contains the following function:

{% highlight swift %}
private func dumpPackageAtExit(_ package: Package, fileNo: Int32) {
    func dump() {
        guard let dumpInfo = dumpInfo else { return }
        let fd = fdopen(dumpInfo.fileNo, "w")
        guard fd != nil else { return }
        fputs(dumpInfo.package.toTOML(), fd)
        for product in products {
            fputs("[[products]]", fd)
            fputs("\n", fd)
            fputs(product.toTOML(), fd)
            fputs("\n", fd)
        }
        fclose(fd)
    }
    dumpInfo = (package, fileNo)
    atexit(dump)
}

{% endhighlight %}

which does two things:

1. Create a function dump that creates TOML representation of current package and write it to given file descriptor
2. Attach this function to atexit hook, so that it is executed when the program exits


> For the curious reader: Sources/PackageLoading/Manifest+parse.swift is also an interesting source file. It contains parse method that actually invokes swiftc to compile Package.swift and get its TOML representation.

## Full Flow

When `swift build` is run,

1. It creates a file descriptor for writing to `.Package.TOML`
2. Invokes `swiftc` in another process to compile the `Package.swift` and run the executable with `-fileno FILE_DESCRIPTOR` argument
3. `Package`’s `init` method is invoked, attaching `dump` method to run `atexit`
4. When the execution is about to end, `TOML` data will be written to the file descriptor
5. SwiftPM parses this `TOML` file and deletes it, that’s why there is no trace of it
