---
layout: post
title:  "PUI: Introduction"
date:   2020-01-18 12:00:00 -0700
categories: pui
---

Annoucing [`pui`](https://crates.io/crates/pui), a library that exoses a set of unqiue identifiers that can be used for a variery of purposes! This series of posts explain how `pui` works, where it can be used, and how you could have developed a similarly generic crate. All posts in this series will be linked in the table of cotents below, and all posts will have a link back here.

## Table of Contents

* [`Introduction`](./PUI-Introduction.html) - <sub><sup> WARNING: recursive link detected </sup></sub>
* [`Part 1: Core`](../20/PUI-Part-1-core) - an introduction to the core mechanics or [`pui`](https://crates.io/crates/pui)

## What is PUI?

[`pui`](https://crates.io/crates/pui) is a library that provides **p**rocess **u**nique **i**dentifiers. What are process unique identifiers (PUIs)? They are values are are **guaranteed** to be unqiue on the process that they reside in. `pui` also provides thread-local unique identifiers, that are unique on the thread they reside in, and can't be moved to another thread.

## Where is PUI?

[`pui`](https://crates.io/crates/pui) is a façade crate that just re-exports it's children:
* [`pui-cell`](https://crates.io/crates/pui-cell)
    * Uses the PUIs to provide safe shared mutable types.
* [`pui-vec`](https://crates.io/crates/pui-vec)
    * Uses PUIs to provide safe unchecked indexing
* [`pui-arena`](https://crates.io/crates/pui-arena)
    * Builds atop `pui-vec` to provide arenas that can do safe unchecked indexing,
      and so much more
* More to come later

We'll be exporing `pui`'s sub-crates in more detail in the following parts of this series.

## Why is PUI?

Unique identifiers are useful because they allow us to elide checks (i.e. array bounds checks), or access shared data in safely. For an example of the latter, see the inspiration for [`pui`](https://crates.io/crates/pui), [`qcell`](https://crates.io/crates/qcell). `qcell` uses these sorts of process unique identifiers (not from `pui`), to verify that it's `*Owner` types can be used to access the `*Cell` types. We'll see how later when we discuss `pui-cell`, which is a reimplementation of `qcell`, but built atop the foundation that `pui` provides.

## How is PUI?

[`pui`](https://crates.io/crates/pui) is very flexible and generic, so generic that it encodes the PUIs behind [`qcell`](https://crates.io/crates/qcell)'s [`TCell`](https://docs.rs/qcell/0.4.1/qcell/struct.TCell.html), [`TLCell`](https://docs.rs/qcell/0.4.1/qcell/struct.TLCell.html), and [`QCell`](https://docs.rs/qcell/0.4.1/qcell/struct.QCell.html) into a single type, `pui::core::Dynamic`! But this flexiblity does come at a small, compile-time cost. In order to simulate `TCell` or `TLCell`, you will need to declare that your types are compatible. You can't just use any old type. But we're getting ahead of ourselves, `pui-core` is supposed the next post!

But the benefits of the generic approach is that it's now useable in a lot more places, and more efficient than the non-generic approach. In particular, `pui-core` doesn't even need an allocator, so it can be used in `#![no_std]` enviornments! `pui-cell` continues this tradition, and is fully compatible with `#![no_std]` environments, so now you can have (mostly) compile time checked shared mutablity even on bare metal! And if you have an allocator, then you can use `pui-vec` and `pui-arena` to get an append only vector or generalized arenas respectively.

See you in the next post, where we dive into [`pui-core`](https://crates.io/crates/pui-core) to see how it provides these process unique identifiers!