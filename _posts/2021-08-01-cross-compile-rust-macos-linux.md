---
layout: post
title: "Building a Rust binary on macOS for Linux!"
category: example
---

## Problem

Recently, I was migrating a discord bot I had previously coded in another language to Rust. The previous version of this bot was hosted by a friend of mine, in a virtualized environment, in Ubuntu. I, on the other hand, was coding this bot on my apple laptop in macOS. Experienced readers will already know where this is going. I, on the other hand, only learned when we tried to run the binary I had built with `cargo build` on the server and got the following message:

```
bash: ./rustybot: cannot execute binary file: Exec format error
```

Oops. It turns out you can't just build a binary on macOS and expect it to run on a Linux system.

So what do we need to do to get our code running on our Linux server?

## Compiler

Before getting into how exactly we solved this problem, I'd like to write a little about what exactly happens when we take a Rust project and compile it into a binary we can run. If you've ever written any Rust, you're probably used to running the command `cargo build` and having it output your binary. What's happening behind the scenes here is RustC, the Rust Compiler, is taking your code and making a binary out of it.

As per the [documentation](https://doc.rust-lang.org/rustc/index.html), RustC is a *cross-compiler* by default. That means you can use it to compile your Rust code to any architecture you want.

Whether we call `rustc` directly or use the `cargo build` command, you can pass an argument as `-target xxxxx`, where xxxxx is our *target triple*. A target triple is something that looks like `x86_64-apple-darwin`, which specifies that we're trying to build a binary for a 64bit macOS system.

Rust ships with a bunch of built-in targets, one of which is `x86_64-unknown-linux-gnu`, so obviously, my first attempt was to add this target with *rustup*[^1] and attempt to build my binary by executing the following commands:

```
rustup add target x86_64-unknown-linux-gnu
cargo build --target=x86_64-unknown-linux-gnu
```

This obviously failed, or this would be a much shorter article.

## Linker

It turns out, building a binary is more complicated than we previously thought. When I added my desired target with the `rustup target add` command, all I did was download the appropriate Rust standard library for that platform. I didn't, however, consider the *Linker*, which we can tell by the following excerpt was the problem.

```
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

The *Linker* is the component that ties together your compiled code to itself and to all the external code it references. Frequently this will be something like the C standard library, which is usually expected to already exist on target systems. 

When we made our last attempt to build our binary, we told the compiler to use the correct Rust standard lib, but we didn't specify which linker to use, so it used the default linker for our macOS system, and that failed.

To solve this issue, we will need to find a linker that runs on macOS for our target platform.

Searching around, I've found two solutions, one of which I'll leave as a suggestion or an exercise to the reader.

1. use [this awesome brew formula](https://github.com/FiloSottile/homebrew-musl-cross) to install a complete compiler toolchain that targets Linux systems
2. use [this other awesome brew formula](https://github.com/SergioBenitez/homebrew-osxct) to install a complete compiler toolchain that targets Linux systems

I realize both of those sound pretty similar. In the end, they are, and for our purposes, both would work. The difference between them is simply one targets [MUSL](https://www.musl-libc.org/), which is an alternative implementation of the C standard library[^2].

For this article, we're going to go with the first option, simply because that's the one I tried first and succeeded with. We can install this brew formula by running

```
brew install FiloSottile/musl-cross/musl-cross
```

We'll also need to add the *MUSL* target to *rustup* so the compiler can use the appropriate rust standard library when compiling with

```
rustup target add x86_64-unknown-linux-musl
```

And then, all we need to do is tell `cargo` to use this specific linker when compiling for this target by creating a file named `config` inside a `.cargo` folder in our project (you're most likely going to need to create this folder). This file should look like this:

```
// .cargo/config

[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"
```

This tells cargo to use the correct linker when compiling for that target.

All this done, we can finally make another attempt at building our binary with

```
cargo build --target=x86_64-unknown-linux-musl
```

### Successssssss 🎉 🥳

```
   Compiling test-linux v0.1.0 (/Users/laurabrehm/dev/cross-compile-playground/test-linux)
    Finished dev [unoptimized + debuginfo] target(s) in 0.93s
```


---
{: data-content="footnotes"}

[^1]: I chose not to dive too deeply into what rustup is in this post, but I promise I'll explain further another time. For now, it suffices to say it can manage rust instalations, as well as it's individual components
[^2]: There are plenty of pros, cons, and differences to be noted regarding [MUSL](https://www.musl-libc.org/), but I didn't think it necessary to further explain and possibly confuse here
