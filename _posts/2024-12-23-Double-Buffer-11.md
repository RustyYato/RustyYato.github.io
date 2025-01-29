---
layout: post
title:  "Double Buffers - Generalized Strategy Part 2"
date:   2024-12-23 12:00:00 -0000
categories: unsafe,advanced rust,library design
---

This is a part of a series of blogs, you can see the full list [here](Double-Buffer-1.html).

Last time we created a `Strategy` trait to generalize the strategies we've made so far, and
to support future strategies. Now let's rework the data structures to use this new trait.

First the double buffer. This will hold the strategy type (as discussed last section) and
will hold the two buffers. For the `Send`/`Sync` bounds, we require the same as in [Part 7](Double-Buffer-7.html).

```rust
struct DoubleBuffer<T, S> {
    strategy: S,
    buffers: [UnsafeCell<T>; 2],
}

// same rationale as in part 7
unsafe impl<T: Send, S: Send> Send for DoubleBuffer<T, S> {}
unsafe impl<T: Send + Sync, S: Sync> Send for DoubleBuffer<T, S> {}
```

And for `WriteHandle`, `ReadHandle` and `ReadGuard`. The handles will
hold onto the buffer and the id specified in the strategy. Since we
own the strategy, and since to create a new `WriterId` you need a `&mut S`
to the strategy, we can ensure that the writer id is not invalidated
for as long as any `WriteHandle` or `ReadHandle` is alive.

For the `ReadGuard`, it will hold onto the handle and to the guard
from the strategy. We will need to take ownership of the guard on drop,
so we need to wrap it in a `ManuallyDrop` to prevent it from being dropped
normally. We also need exclusive access to the handle so that we can
pass a `&mut S::ReaderId` to the strategy on drop.

```rust
struct WriteHandle<T, S: Strategy> {
    id: S::WriterId,
    inner: Arc<DoubleBuffer<T, S>>
}

struct ReadHandle<T, S: Strategy> {
    id: S::ReaderId,
    inner: Arc<DoubleBuffer<T, S>>
}

struct ReadGuard<'a, T, S: Strategy> {
    guard: ManuallyDrop<S::ReadGuard>,
    handle: &'a mut ReadHandle<T, S>
}
```

Now we have the data structures in place, add constructors for the handles.
Since we know that the `WriteHandle` and `ReadHandle` is a **valid writer** and
a **valid reader**, we can discharge the safety requirements of `create_reader_id`.

It is important that we never expose `&mut S` to the user, since that would be the
only way to invalidate this requirement.

```rust
impl<T, S: Strategy> WriteHandle<T, S> {
    pub fn new(mut strategy: S, buffers: [T; 2]) -> Self {
        let id = strategy.create_writer_id();

        Self {
            id,
            inner: Arc::new(DoubleBuffer {
                strategy,
                buffers: buffers.map(UnsafeCell::new),
            })
        }
    }

    pub fn read_handle(&self) -> ReadHandle<T, S> {
        // SAFETY: `WriteHandle` always holds onto a valid writer id
        let id = unsafe { self.inner.strategy.create_reader_id(ReaderOrWriter::Writer(&self.id)) };

        ReadHandle {
            id,
            inner: self.inner.clone(),
        }
    }
}

impl<T, S: Strategy> Clone for ReadHandle<T, S> {
    fn clone(&self) -> Self {
        // SAFETY: `ReadHandle` always holds onto a valid reader id
        let id = self.inner.strategy.create_reader_id(ReaderOrWriter::Reader(&self.id));

        ReadHandle {
            id,
            inner: self.inner.clone(),
        }
    }
}
```

Now for swapping buffers and creating/dropping a read guard, just call the respective methods on the strategy.

```rust
impl<T, S: Strategy> WriteHandle<T, S> {
    /// # Safety
    /// 
    /// A swap must be run to completion before mutable accessing the buffer
    unsafe fn try_start_swap(&mut self) -> Result<SwapInfo<S>, S::SwapError> {
        let swap = self.inner.strategy.try_start_swap(&mut self.id)?;
        Ok(SwapInfo {
            swap
        })
    }

    /// # Safety
    /// 
    /// This must be the latest swap created by `try_start_swap`
    unsafe fn finish_swap(&mut self, swap: SwapInfo<S>) {
        self.inner.strategy.finish_swap(&mut self.id, swap)
    }

    pub fn swap_buffers(&mut self) -> Result<(), S::SwapError> {
        // SAFETY: this swap will be run to completion just below
        let swap = unsafe { self.try_start_swap()? };
        // SAFETY: this swap was just created above
        unsafe { self.finish_swap(swap) }
        Ok(())
    }
}

impl<T, S> ReadHandle<T, S> {
    pub fn read(&mut self) -> ReadGuard<'_, T, S> {
        // SAFETY: `ReadHandle` always holds onto a valid reader id
        let guard = unsafe { self.inner.strategy.acquire_read_guard(&mut self.id) };
        ReadGuard {
            guard,
            handle: self
        }
    }
}
```

For dropping the `ReadGuard` we need to extract the `S::ReadGuard` type. But since
`Drop` only gives us a `&mut self`, we can't do that normally. Instead we will utilize
`ManuallyDrop::take` to take ownership of the `S::ReadGuard`. `ManuallyDrop` will prevent
double-drops in the drop-glue, so we are safe.

```rust
impl<T, S> Drop for ReadGuard<'_, T, S> {
    fn drop(&mut self) {
        let guard = ManuallyDrop::take(&mut self.guard);
        // SAFETY: `ReadHandle` always holds onto a valid reader id
        //         `ReadGuard` always holds onto a valid guard
        unsafe { self.handle.inner.strategy.release_read_guard(&mut self.handle.id, guard); }
    }
}
```

Finally we need to add the accessors again.

```rust
impl<T, S> DoubleBuffer<T, S> {
    fn read_buffer(&self) -> &T {
        unsafe { &*self.handle.data.get().cast::<T>().add((!self.which) as usize) }
    }

    /// # Safety
    /// 
    /// Only the writer is allowed to access this buffer
    unsafe fn write_buffer(&self) -> &T {
        unsafe { &*self.handle.data.get().cast::<T>().add(self.which as usize) }
    }

    /// # Safety
    /// 
    /// Only the writer is allowed to access this buffer
    unsafe fn write_buffer_mut(&self) -> &mut T {
        unsafe { &mut *self.handle.data.get().cast::<T>().add(self.which as usize) }
    }
}

impl<T, S: Strategy> WriteHandle<T, S> {
    pub fn read_buffer(&self) -> &T {
        self.inner.read_buffer()
    }

    pub fn write_buffer(&self) -> &T {
        // SAFETY: we are the writer
        unsafe { self.inner.write_buffer() }
    }

    pub fn write_buffer(&mut self) -> &mut T {
        // SAFETY: we are the writer
        unsafe { self.inner.write_buffer() }
    }
}

impl<T, S: Strategy> Deref for ReadGuard<T, S> {
    type Target = T;

    fn deref(&self) -> &T {
        self.handle.inner.read_buffer()
    }
}
```

This now concludes the basic API, but we also had `DelayWriteHandle` to make starting and
finishing swaps safe, and to amortize the cost of waiting for readers to exit the buffer.
Let's port that over as well.

```rust
struct DelayWriteHandle<T, S: Strategy> {
    swap: Option<S::SwapInfo>,
    handle: WriteHandle<T, S>,
}

impl<T, S> DelayWriteHandle<T, S> {
    pub fn new(handle: WriteHandle<T, S>) -> Self {
        Self {
            swap: None,
            handle,
        }
    }

    pub fn get(&self) -> &WriteHandle<T, S> {
        &self.handle
    }

    pub fn start_swap(&mut self) {
        if self.swap.is_none() {
            // SAFETY: this swap will be finished before accessing the buffers via exclusive references
            self.swap = Some(unsafe { self.handle.start_swap() })
        }
    }

    pub fn finish_swap(&mut self) ->  &mut WriteHandle<T, S> {
        if let Some(swap) = self.swap.take() {
            // SAFETY: this was the last swap, and it was created by this handle
            unsafe { self.handle.finish_swap(swap) }
        }

        &mut self.handle
    }
}
```

With this the core API is now complete. I've leave it to you to port `OpWriteHandle` and hashmap impl.
Next time we will investigate the new strategy for synchronization which fixes one of the largest downsides
of `evmap`'s strategy: `swap_buffers`.