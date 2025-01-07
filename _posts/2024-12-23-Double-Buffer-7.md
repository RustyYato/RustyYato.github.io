---
layout: post
title:  "Double Buffers - Concurrent Updates Part 2"
date:   2024-12-23 12:00:00 -0000
categories: unsafe,advanced rust,library design
---

Let's make a production grade multi-threaded double buffer.

This is a part of a series of blogs, you can see the full list [here](Double-Buffer-1.html).

[Previously](Double-Buffer-5.html) we implemented a sound double buffer, but it didn't have the
performance characteristics that we set out to implement. In particular, readers didn't scale independently
of writers, and could be blocked indefinitely by writers. To fix this, we will adapt the synchronization strategy
used by [evmap](https://crates.io/crates/evmap/10.0.2) made by [@jonhoo](https://www.youtube.com/@jonhoo). This isn't
a 1-1 implementation of `evmap`'s synchronization strategy, but it is heavily inspired by it, and mostly only
differs in the details of the implementation.

Before jumping into the code, let's get the high-level ideas down first. 

## `evmap` synchronization strategy theory

The double buffer will store a `flag: AtomicBool`, which signals which buffer is currently active.

### Readers

Each reader of the double buffer will manage a shared counter (`Arc<AtomicUsize>`) called an `epoch`.
This `epoch` carry two pieces of information:
 1. If a reader is currently reading from the buffer
 2. How many times the reader read from the buffer (their `count`)

When a readers starts reading from the buffer it will:
 1. increment the `count`
 2. load the `flag` to see which buffer it's pointing into

When a reader finishes reading from the buffer it will:
 1. increment the `count`
 2. notify the writer that this reader has finished reading

When a reader's `count` is odd, then it is currently reading, if it is `odd` it is inert.

### Writers

In the shared double buffer, we will hold a list of these shared counters, so the writer may access the counts as well.

Swapping will consist of two phases: a setup and an iterative phase.

During the setup phase, the writer will
1. swap the buffers
2. garbage collect any readers which are no longer alive
3. see which readers are still reading from the buffer, and store their `count` in a second buffer

During the iterative phase, the writer will iterate over all readers which where reading during the setup phase (in the second buffer)
1. if their `count` is the same as in the second buffer, then they are still reading from the buffer.
    Restart the loop, and wait until a reader finishing reading.
2. if their `count` is different from before, then they are not reading from the buffer.

The iterative phase ends when the the writer has seen that all readers are no longer reading from the map at some point during
the iterative phase.

Before going into the implementation proper, this description should be specific enough to write you own implementation
of this strategy, so try to do it! Then check back to see how it compares with what I describe below. What's different,
what's the same. What assumptions did you make? Can you prove the correctness of your implementation? What `Ordering`s did you use?

## The implementation

Now let's see how this looks in practice!
