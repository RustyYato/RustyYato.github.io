---
layout: post
title:  "Double Buffers - Concurrent Updates Part 1"
date:   2024-12-23 12:00:00 -0000
categories: unsafe,advanced rust,library design
---

Time to convert our single-threaded double buffer to a multi-threaded double buffer.

This is a part of a series of blogs, you can see the full list [here](Double-Buffer-1.html).

## A Multithreaded implementation

In our previous section we used a double buffer like below, but this is clearly not thread-safe.
So we need to update this to reach our goal of a concurrent hash map.

```rust
use std::cell::{Cell, UnsafeCell};
use std::rc::Rc;

struct DoubleBuffer<T> {
    which: Cell<bool>,
    active_readers: Cell<u32>,
    // we use UnsafeCell here because we are formally sharing the data, even if they are all accesses are disjoint
    data: [UnsafeCell<T>; 2],
}

pub struct WriteHandle<T> {
    buffer: Rc<DoubleBuffer<T>>,
}

pub struct ReadHandle<T> {
    buffer: Rc<DoubleBuffer<T>>,
}
```

So how do we fix this? We could try naively switching to atomics. Let's try it out and see how it goes

```rust
use std::sync::atomic::{AtomicBool, AtomicU32, Ordering};
use std::sync::Arc;

struct DoubleBuffer<T> {
    which: AtomicBool,
    active_readers: [AtomicU32; 2],
    // we use UnsafeCell here because we are formally sharing the data, even if they are all accesses are disjoint
    data: [UnsafeCell<T>; 2],
}

pub struct WriteHandle<T> {
    buffer: Arc<DoubleBuffer<T>>,
}

pub struct ReadHandle<T> {
    buffer: Arc<DoubleBuffer<T>>,
}
```

Now we did need to make one more change, `active_readers` is now two counters instead of one. This allows us to track how many
readers are in each buffer.

Then we can simply increment the counter for the buffer we are looking at.

```rust
impl WriteHandle<T> {
    // -- snip --

    pub fn try_swap_buffers(&mut self) -> bool {
        let next_which = !self.buffer.which.load(Ordering::Relaxed);
        
        if self.buffer.active_readers[next_which].load(Ordering::Acquire) == 0 {
            self.buffer.which.store(next_which, Ordering::Release);
            true
        } else {
            false
        }
    }

    pub fn swap_buffers(&mut self) {
        assert!(self.try_swap_buffers(), "Tried to swap buffers while there was an active reader")
    }

    // -- snip --
}

impl<T> ReadHandle<T> {
    pub fn read_buffer(&self) -> ReadGuard<'_, T> {
        let which = self.buffer.which.load(Ordering::Acquire);
        let old_readers = self.buffer.active_readers[which as usize].fetch_add(1, Ordering::Release);
        if old_readers == u32::MAX {
            panic!("Tried to read too many times")
        }
        ReadGuard {
            which,
            buffer: &self.buffer,
        }
    }
}
```

The `ReadGuard` has the same representation from last time, but we will need to update the `Drop` impl to account for the two counters
and atomics.

```rust
struct ReadGuard<'a, T> {
    which: bool,
    buffer: &'a DoubleBuffer<T>,
}

impl<T> Deref for ReadGuard<'_, T> {
    type Target = T;

    fn deref(&self) -> &T {
        let buffer = &self.buffer.data[(!self.which) as usize];
        // SAFETY: We ensure that the writer isn't allowed to access this data
        unsafe { &*buffer.get() }
    }
}

// this has changed
impl<T> Drop for ReadGuard<'_, T> {
    fn drop(&mut self) {
        let readers = &self.buffer.active_readers[(!self.which) as usize];
        readers.fetch_sub(1, Ordering::Release);
    }
}
```

And again if we try to try to swap two buffers while there is an active reader, we get a panic.
This is worse for a multi-threaded double buffer, but at least this is thread-safe. Right?

```rust
let mut writer = WriteHandle::new([0, 1]);
let reader = writer.read_handle();

let a = reader.read_buffer();
writer.swap_buffers();
let b = writer.write_buffer();
assert_eq!(*a, *b); // now unreachable
```

```text
thread 'main' panicked at src/main.rs:42:9:
Tried to swap buffers while there was an active reader
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

## Thread Safety

```rust
// -- snip --

// in ReadHandle::read_buffer

let which = self.buffer.which.load(Ordering::Acquire);
let old_readers = self.buffer.active_readers[which as usize].fetch_add(1, Ordering::Release);
if old_readers == u32::MAX {
    panic!("Tried to read too many times")
}

// -- snip --

// in WriteHandle::try_swap_buffers

let next_which = !self.buffer.which.load(Ordering::Relaxed);

if self.buffer.active_readers[next_which].load(Ordering::Acquire) == 0 {
    // point A

    self.buffer.which.store(next_which, Ordering::Release);
    true
} else {
    false
}

// -- snip --
```

Above is how we just defined reading and swapping buffers. But this does not uphold the guarantees that we want.
In particular, if `ReadHandle::read_buffer` is running on `Thread 1` and `WriteHandle::try_swap_buffers` is running on `Thread 2`,
we could have the following code execution:
 * run `WriteHandle::try_swap_buffers` in `Thread 2` until point A
 * run `ReadHandle::read_buffer` to completion in `Thread 1`
 * finish `WriteHandle::try_swap_buffers` in `Thread 2`

This will lead to both the `ReadHandle` in `Thread 1` and the `WriteHandle` in `Thread 2` both pointing to the same buffer.
So our API can expose an aliased `&mut _`, which makes this API *unsound*.

So it looks like just naively translating our single-threaded code to the multi-threaded implementation doesn't work. In general,
single-threaded algorithms fundamentally rely on being single-threaded, so cannot be easily translated to a multi-threaded
implementation naively.

## A Sound Multithreaded implementation

So to fix this, we will need to introduce a loop and a lock.

```rust
impl WriteHandle<T> {
    // -- snip --

    pub fn try_swap_buffers(&mut self) -> bool {
        let next_which = !self.buffer.which.load(Ordering::Relaxed);
        
        // before swapping, lock the current readers in the next buffer. If there are already any readers, then we will fail to lock
        if self.buffer.active_readers[next_which].compare_exchange(0, u32::MAX, Ordering::AcqRel, Ordering::Acquire).is_ok() {
            // under the lock, swap the buffers
            self.buffer.which.store(next_which, Ordering::Release);
            // release the lock
            self.buffer.active_readers[next_which].store(0, Ordering::Release);
            true
        } else {
            false
        }
    }

    pub fn swap_buffers(&mut self) {
        assert!(self.try_swap_buffers(), "Tried to swap buffers while there was an active reader")
    }

    // -- snip --
}

impl<T> ReadHandle<T> {
    pub fn read_buffer(&self) -> ReadGuard<'_, T> {
        let mut which = self.buffer.which.load(Ordering::Acquire);
        let mut readers = &self.buffer.active_readers[(!swapped) as usize];

        let mut old_readers = readers.load(Ordering::Acquire);

        loop {
            // try to increment the counter
            let Some(next_num_readers) = num_readers.checked_add(1) else {
                // if it is locked by the `WriteHandle`, then refresh
                // all pointers, and point into the next buffer.    
                which = self.buffer.which.load(Ordering::Acquire);
                reader_count = &self.num_readers[(!swapped) as usize];
                num_readers = reader_count.load(Ordering::Acquire);

                core::hint::spin_loop();
                continue;
            };

            // try to update `reader_count`
            match reader_count.compare_exchange_weak(
                num_readers,
                next_num_readers,
                Ordering::AcqRel,
                Ordering::Acquire,
            ) {
                Err(current) => num_readers = current,
                Ok(_) => return ReadGuard {
                    which,
                    buffer: &self.buffer,
                },
            }

            core::hint::spin_loop();

        }
    }
}
```

The write locks the `active_readers` for the next buffer, then does the swap. This ensures that all readers that are racing with
`WriteHandle::try_swap_buffer` are aware that a buffer swap is taking place, and can cooperate (by reloading which buffer they should
point to). Now, this implementation is still flawed, it ties read scaling back to the writer. And if a writer is constantly swapping,
then readers will be unable to make progress. So this is not ideals. But it is *sound*, which is far better than we can say about the
initial implementation.