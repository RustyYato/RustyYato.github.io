---
layout: post
title:  "Double Buffers - Generalized Strategy"
date:   2024-12-23 12:00:00 -0000
categories: unsafe,advanced rust,library design
---

This is a part of a series of blogs, you can see the full list [here](Double-Buffer-1.html).

Last time we created a concurrent hash map from the double buffer. But
there's a better strategy for synchronization from [`flashmap`](https://crates.io/crates/flashmap).
Also, it would be nice if we could support the single-threaded strategies as well.
However, I don't want to rewrite the entire crate for each strategy.

So now it's time to generalize the work we've done to support multiple strategies.
Let's take a look at [Part 7](Double-Buffer-7.html), which was the more complex strategy
we've made. We'll fill in this `Strategy` trait as we go along.

The types that implement this  `Strategy` trait will hold the data we normally put in `DoubleBuffer`.
And since we are relying on implementations of `Strategy` being correct to prevent data-races,
we need to make this an unsafe trait.

I won't include the documentation here for brevity.

```rust
unsafe trait Strategy {
    // TODO
}
```

We will nee a few associated types
* What data to be stored in the `WriteHandle`
    * for example, the `last_epochs` list to amortize the cost of allocations
* What data to be stored in the `ReadHandle`
    * for example, the `epoch` field to avoid the cost of locking on every read
* What data to be stored in the `ReadGuard`
    * for example, the `which: bool` flag that specified which buffer to read from
* what data to needs to be kept around during a `swap`
    * for example, the `start: usize` field to allow skipping past readers that were already checked
* what error to raise if swap failed
    * for example, in our single-threaded case, it's better to just fail instead of block

```rust
unsafe trait Strategy {
    type WriterId;
    type ReaderId;
    type ReadGuard;

    type SwapInfo;
    type SwapError;

    // TODO
}
```

Next we will need some constructors for reader and writer ids.
We need to ensure that there is only one writer id, at a given time.
And that we don't use old reader or writer ids.

To do this, we are going to introduce the notion of a **valid reader** and
**valid writer**. A **valid writer** is the last writer id that was created.
Any previous writer ids get invalidated whenever a new writer id is created.

A **valid reader** is a reader that is created from a **valid writer** or
another **valid reader**. A reader id `A` is **created from** a writer id `W` if
1. the `W` was used to construct `A`
2. the reader id `B` was used to construct `A`, and `B` is **created from** `W`

A reader id is invalidated when the writer id it was **created from** is invalidated.

It is undefined behavior to try to construct a reader from an invalid writer.

```rust
enum ReaderOrWriter<'a, R, W> {
    Reader(&'a R),
    Writer(&'a W),
}

trait Strategy {
    // ... SNIP ...

    fn create_writer_id(&mut self) -> Self::WriterId;

    unsafe fn create_reader_id(&self, source: ReaderOrWriter<'_, Self::ReaderId, Self::WriterId>) -> Self::ReaderId;

    // TODO
}
```

Now let's handle the major writer methods for swapping the buffers.
Each of these methods will need to take a **valid writer** to ensure correctness.

We will need a setup phase, a way to check if a given swap is finished, and
a method to wait until the swap is finished.

`is_swap_finished` will only guarantee correctness if the swap is the last swap that was created.
But we should't cause UB if it wasn't the last swap.

```rust
// ... SNIP ...

trait Strategy {
    // ... SNIP ...
    
    unsafe trait try_start_swap(&self, writer: &mut Self::WriterId) -> Result<Self::SwapInfo, Self::SwapError>;

    unsafe fn is_swap_finished(&self, writer: &mut Self::WriterId, &mut Self::SwapInfo) -> bool;

    unsafe fn finish_swap(&self, writer: &mut Self::WriterId, Self::SwapInfo);

    // TODO
}
```

Onwards, to the reader, we need two methods, one to acquire the read guard and one to release it.
Just like we saw in [Part 7](Double-Buffer-7.html). This way the user can keep track of which buffer to read.

```rust
// ... SNIP ...

trait Strategy {
    // ... SNIP ...
    
    unsafe trait acquire_read_guard(&self, reader: &mut Self::ReaderId) -> Self::ReadGuard;
    
    unsafe trait release_read_guard(&self, reader: &mut Self::ReaderId, guard: Self::ReadGuard);

    // TODO
}
```

Finally we need some accessors for which buffer to read. We need two methods for this, one for the
writer and one for the reader because the reader will need to access the `ReadGuard`, but the writer
doesn't need to (and in fact can't).

```rust
// ... SNIP ...

trait Strategy {
    // ... SNIP ...
    
    unsafe trait is_swapped_writer(&self, writer: &Self::WriterId) -> bool;
    
    unsafe trait is_swapped_reader(&self, reader: &mut Self::ReaderId, guard: &Self::ReadGuard) -> bool;
}
```

With this interface we can create a generic version of `WriteHandle`, `ReadHandle`, and `ReadGuard`.