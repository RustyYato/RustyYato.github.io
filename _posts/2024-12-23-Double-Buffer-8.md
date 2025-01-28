---
layout: post
title:  "Double Buffers - Delayed Swaps"
date:   2024-12-23 12:00:00 -0000
categories: unsafe,advanced rust,library design
---

Let's implement an important optimization to reduce the overhead of swaps.

This is a part of a series of blogs, you can see the full list [here](Double-Buffer-1.html).

In [Part 7](Double-Buffer-7.html), we implemented a concurrent double buffer which can handle multiple readers on many threads.
However, we sacrificed the performance of writes to do so. Now writes can take arbitrarily long on buffer swaps, since it
has to wait for all readers to stop reading from the buffer. But we can do better.

Taking another page from [`evmap`](https://crates.io/crates/evmap/), and we can see what we need to do.
The key insight here is that readers won't hold on to their lock forever, and when the re-acquire the lock after a swap,
they will acquire a lock into the new buffer. We can exploit this little bit of wiggle room.

Right now, we do something like this on swaps:

1. swap buffers
2. wait for readers
3. allow access to buffers

All of this happening inside a single function. But what if instead of we did something like this:

1. wait for readers from previous swap
2. allow access to buffers
3. swap buffers

Small problem... how do we expose access to the buffers? It would be nice if we could split these up some more.
Maybe it's time for a new `WriteHandle` to expose a slightly different API?

```rust
pub struct DelayWriteHandle<T> {
    swap: Option<SwapInfo>,
    handle: WriteHandle<T>
}

impl<T> DelayWriteHandle<T> {
    pub fn new(handle: WriteHandle<T>) -> Self {
        Self {
            swap: None,
            handle,
        }
    }

    pub fn get(&self) -> &WriteHandle<T> {
        &self.handle
    }

    pub fn start_swap(&mut self) {
        if self.swap.is_none() {
            // SAFETY: this swap will be finished before accessing the buffers via exclusive references
            self.swap = Some(unsafe { self.handle.start_swap() })
        }
    }

    pub fn finish_swap(&mut self) ->  &mut WriteHandle<T> {
        if let Some(swap) = self.swap.take() {
            // SAFETY: this was the last swap, and it was created by this handle
            unsafe { self.handle.finish_swap(swap) }
        }

        &mut self.handle
    }
}
```

The key point of this API is that we can't freely access the an exclusive reference to the `WriteHandle`, and
thus we can't freely access the underlying buffers. It's possible to get shared access to the `WriteHandle`,
and this is fine, since we are allowed to get shared access the buffers.

We can only get exclusive access to the `WriteHandle` after proving that there are no readers currently reading
from the buffer, by calling `finish_swap`. While this is a small addition, this unlocks a brave new world.
We now have all the tools needed to write a concurrent hashmap that rivals [`evmap`](https://crates.io/crates/evmap/). 
And the concurrent hashmap will not require *any* new unsafe code!

But before we go ahead and do that, let's implement more more feature (no unsafe code required). [`evmap`](https://crates.io/crates/evmap/)
uses an operation log to handle all of it's operations. This way it can replay operations exactly on both buffers.
Let's make a writer which can handle this for us.

First off we need to specify operations, in a general way so this is usable outside a concurrent hashmap.

```rust
/// An operation which can be applied to a buffer of type `T`
/// 
/// This operation should make the same mutations to the buffer,
/// if two equal buffers are given
pub trait Operation<T>: Sized {
    fn apply(&mut self, buffer: &mut T);

    // an optimization to allow consuming the operation on the second use
    fn apply_once(mut self, buffer: &mut T) {
        self.apply(buffer)
    }
}
```

This is almost like a closure, but making it our own trait let's the users implement this on their our named types,
and it allows us to implement the `apply_once` optimization. 

Now for the upgraded writer.

```rust
struct OpWriteHandle<T, O> {
    waterline: usize,
    ops: Vec<O>,
    handle: DelayWriteHandle<T>,
}

impl<T, O> OpWriteHandle<T, O> {
    pub fn new(handle: DelayWriteHandle<T>) -> Self {
        Self {
            waterline: 0,
            ops: Vec::new(),
            handle,
        }
    }

    pub fn get(&self) -> &WriteHandle<T> {
        self.handle.get()
    }

    pub fn push(&mut self, op: O) {
        self.ops.push(op);
    }
}
```

We store a list of operations to apply on the next swap, and a waterline (which signifies which operations have already been applied).

```rust
impl<T, O: Operation> OpWriteHandle<T, O> {
    pub fn swap_buffers(&mut self) -> Self {
        // finish the previous swap
        let handle = self.handle.finish_swap();
        // get the buffer, this will be a different buffer from the last swap
        let buffer = handle.get_write_buffer_mut();

        // apply all operations that were applied to the last buffer
        // to keep the two buffers in sync
        for o in self.ops.drain(..self.waterline) {
            o.apply_once(buffer);
        }

        // set the new waterline to the operations we are about to apply
        self.waterline = self.ops.len();

        // apply all new operations to this buffer
        for o in &mut self.ops {
            o.apply(buffer);
        }

        // swap the buffers to expose the new operations to the readers
        self.handle.start_swap();
    }
}
```

This scheme ensures that both buffers stay in sync, by applying the same operations to each of them in the same order.
We never expose an exclusive reference to the handle to the user, since if we did that we couldn't guarantee that
the two buffers stay in sync. But since there's no unsafe code at this stage, even if you used a malicious
operation, you couldn't trigger undefined behavior.

With this we now have all the required tools to build the concurrent hashmap. Join us next time to build that out!