---
layout: post
title:  "Advanced Rust Library Design - from Double Buffers to a Concurrent HashMap"
date:   2024-12-23 12:00:00 -0000
categories: unsafe,advanced rust,library design
---

In this series we will implement a double buffer library, similar to [`left-right`](https://crates.io/crates/left-right) but a lot more general.

Here is a preview of what is to come
 * [**Humble Beginnings**](Double-Buffer-2.html) - see how double buffers are used, figure out use-cases, and implement a straight-forward single-threaded double-buffer
 * [**A Fresh Start**](Double-Buffer-3.html) - use what we learned in **Humble Beginnings** to implement a better to use double buffer library
 * [**Interlude on Unsafe**](Double-Buffer-4.html) - explore how unsafe interacts with various parts of Rust, what are safety contracts more
 * [**Concurrent Updates Part 1**](Double-Buffer-5.html) - adapt the single-threaded library to an naive consistent multi-threaded double buffer
 * [**Interlude on Atomics**](Double-Buffer-6.html) - convert the naive double buffer to an eventually consistent multi-threaded double buffer
 * [**Concurrent Updates Part 2**](Double-Buffer-7.html) - convert the naive double buffer to an eventually consistent multi-threaded double buffer
 * **Generalizing Part 1** - generalize the library to handle custom pointers, and make it `no_std` compatible
 * **Generalizing Part 2** - generalize the strategy used to synchronize the two buffers
 * [**Delayed Swaps**](Double-Buffer-8.html) - implement an important optimization to reduce the overhead of swaps
 * **A Better Strategy** - implement a better strategy designed by [`Cassy343`](https://github.com/Cassy343) in [`flashmap`](https://crates.io/crates/flashmap)
 * **Async Compatibility** - extend support to allow async buffer swaps
 * [**Concurrent HashMap**](Double-Buffer-8.html) - implement a concurrent, eventually consistent, hashmap with a completely safe interface (including `retain`!)
 * **Subtle Tricks** - explore some tricks to regain some more power out of our double buffer library

At the end of each section, I will leave you with some questions. These will usually be covered in the
next section, but it will be good for you to work them out yourself, cause that's where you'll learn.