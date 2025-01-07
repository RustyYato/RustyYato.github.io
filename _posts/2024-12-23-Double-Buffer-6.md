---
layout: post
title:  "Double Buffers - Interlude on Atomics"
date:   2024-12-23 12:00:00 -0000
categories: unsafe,advanced rust,library design
---

A crash course on atomics!

This is a part of a series of blogs, you can see the full list [here](Double-Buffer-1.html).

[Last time](Double-Buffer-5.html) we implemented a sound double buffer, and we had to dip into
atomics to get a similar implementation to our single-threaded implementation. I was a bit hand-wavey
about it, so now it's time to dig into atomics and how to work with them.

This will be a crash course, I will only be explaining just enough to get going. If you need a more
in depth look at atomics, how they are implemented in hardware, and a more in-depth explanation on
how to use them, check out Mara Bos's [Rust Atomics and Locks](https://marabos.nl/atomics/) book!
(It's available online for free, but I highly recommend you buy it to support their work!).
If you do end up reading that, you can safely skip this section and move onto the [next one](Double-Buffer-7.html), where
we'll get our hands dirty with a production grade implementation of multi-threaded double buffers.

#TODO - Write the atomics section