---
layout: page
title: About Me
permalink: /about/
---

I work on open source crates and am active on
* [Rust Users](https://users.rust-lang.org/u/RustyYato/summary)
* [Rust Internals](https://internals.rust-lang.org/u/RustyYato/summary)
* [Github](https://github.com/RustyYato)
* [Reddit](https://www.reddit.com/user/YatoRust)

This blog is about the Rust Programming Language. Here we will delve into the details of how Rust operates, how it desugars various constructs, and more!

I have created a number of crates including:
 * [`generic-field-projection`](https://github.com/RustyYato/generic-field-projection) - A prototype implementation for [RFC #2708](https://github.com/rust-lang/rfcs/pull/2708), which allows you overload the dot operator to get field access on arbitrary types.
    * [`ptr-to-field`](https://github.com/RustyYato/ptr-to-field) was a precursor to `generic-field-projection` and helped inform it's design decisions.
    * [`typsy`](https://github.com/RustyYato/typsy) - Type level list and enums which is used in the internals of `generic-field-projection`. This crate was heavily inspired by [`frunk`](https://crates.io/crates/frunk)
 * [`double-buffer`](https://github.com/RustyYato/double-buffer) - A reimplementation of [`left-right`](crates.io/crates/left-right) that has a few different design decisions and serves as a lower level api to `left-right`
 * [`generic-vec`](https://github.com/RustyYato/generic-vec) – a `Vec` implementation that is generic over its storage strategy
    * For example, it's possible to implement both a heap-backed vector and an array backed vector with this library and use `GenericVec` without caring about how its data is stored
 * [`type-families`](https://github.com/RustyYato/type-families) - An implementation of a few popular type-classes from Haskell in stable Rust
 * [`rel-ptr`](https://github.com/RustyYato/rel-ptr) - Relative pointers in Rust, which can be used to soundly create one class of self-referenential types.
 * [`vec-utils`](https://github.com/RustyYato/vec-utils) - Extra functions to effiencently covnert a `Vec<T>` to `Vec<U>`, reusing the memory where possible
 * [`set_slice`](https://github.com/RustyYato/published_crates/tree/master/set_slice) - My first crate, some syntactic sugar to allow setting arbitrary sub-slices of a slice

I've also work on a number of open source projects, chief among them:
* Rust `std` library [`PR`](https://github.com/rust-lang/rust/pull/56796) - where I modified the blanket implementations on `From -> TryFrom` to `Into -> TryFrom`
* MIRI deallocation hooks [`PR 1`](https://github.com/rust-lang/miri/pull/1334), [`PR 2`](https://github.com/rust-lang/rust/pull/70962) - where I added deallocation hooks to MIRI, which shows you when an allocation is deallcoated when it is tracked.
* Made [`once_cell`](https://github.com/matklad/once_cell) more flexible [`PR`](https://github.com/matklad/once_cell/pull/37) - Changed `once_cell::*::Lazy` to allow `FnOnce` closures instead of just `Fn` closures
   * `once_cell` is now in the process of getting merged into std, which is exciting
* Exposed useful traits in [`packed_simd`](https://github.com/rust-lang/packed_simd) [`PR`](https://github.com/rust-lang/packed_simd/pull/259)
* Found subtle soundness holes in multiple crates:
   * in [`dashmap`](https://github.com/xacrimon/dashmap) - [soundness hole](https://github.com/xacrimon/dashmap/issues/10), [PR](https://github.com/xacrimon/dashmap/pull/11)
   * in [`evmap`](https://github.com/jonhoo/left-right) - [soundness hole](https://github.com/jonhoo/left-right/issues/75)
   * in [`stowaway`](https://github.com/Lucretiel/stowaway/) - [soundness hole](https://github.com/Lucretiel/stowaway/issues/8)