---
layout: page
title: About Me
permalink: /about/
---

This blog is about the Rust Programming Language. Here we will delve into the details of how Rust operates, how it desugars various constructs, and more! This blog will assume that you have read [The Rust Programming Language](https://doc.rust-lang.org/book/).

I am `RustyYato`, and I am active on
* [Rust Users](https://users.rust-lang.org/u/RustyYato/summary)
* [Rust Internals](https://internals.rust-lang.org/u/RustyYato/summary)
* [Github](https://github.com/RustyYato)
* [Reddit](https://www.reddit.com/user/YatoRust)

These blog posts will be posted to the Rust Users Forum and Reddit whenever I make something new.

I have created a number of crates including:
 * [`rel-ptr`](https://github.com/RustyYato/rel-ptr) - Relative pointers in Rust, which can be used to soundly create one class of self-referenential types.
 * [`vec-utils`](https://github.com/RustyYato/vec-utils) - Extra functions to effiencently covnert a `Vec<T>` to `Vec<U>`, reusing the memory where possible
 * [`generic-field-projection`](https://github.com/RustyYato/generic-field-projection) - A prototype implementation for [RFC #2708](https://github.com/rust-lang/rfcs/pull/2708), which allows you overload the dot operator to get field access on arbitrary types.
    * [`ptr-to-field`](https://github.com/RustyYato/ptr-to-field) was a precursor to `generic-field-projection`, and helped inform it's design decisions.
    * [`typsy`](https://github.com/RustyYato/typsy) - Type level list and enums which is used in the internals of `generic-field-projection`. This crate was heavily inspired by [`frunk`](https://crates.io/crates/frunk)
 * [`generic-vec`](https://github.com/RustyYato/generic-vec) – a `Vec` implementation that is generic over its storage strategy
    * For example, it's possible to implement both a `Heap`-backed vector and an `Array` backed vector with this library, and use `Vec` without caring about how its data is stored
 * [`type-families`](https://github.com/RustyYato/type-families) - An implementation of some popular type-classes from `Haskell` in stable `Rust`
 * [`set_slice`](https://github.com/RustyYato/published_crates/tree/master/set_slice) - My first crate, some syntactic sugar to allow setting arbitrary sub-slices of a slice
