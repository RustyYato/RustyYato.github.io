---
layout: post
title:  "Double Buffers - A Fresh Start"
date:   2024-12-23 12:00:00 -0000
categories: unsafe,advanced rust,library design
---

This is a part of a series of blogs, you can see the full list [here](Double-Buffer-1.html).

In the last section we learned what double buffers are and some features a double buffer should have.
 1. updates and reads are should be independent of each other, this will allow us to scale reads
 2. swapping buffers shouldn't force users to explicitly re-acquire the buffers, this is just bad UX
 3. the library should manage the lifetime of the double buffer and provide some sharing, so it is easy to achieve the previous goals

We may find some other goals/features we want our double buffer to have, but this is a good start for now.

Something I couldn't spoil in the last part is why part 3 is so important.
Point 3 is also visible in other data structures, for example channels are usually split into sender and receiver
halves that the user is supposed to manage. For a mpsc channel is used so that you can clone the producers, and
send them to different threads. Since this double buffer will eventually be used to build a concurrent hash map,
and readers of that hash map should be clone-able it will be helpful to split they up readers and writes.

To that end, the approach we'll take is by hiding the double buffer behind a pointer, and giving the user a write handle and a read handle.
These handles will be thin wrappers around those pointers.

```rust
use std::cell::{Cell, UnsafeCell};
use std::rc::Rc;

struct DoubleBuffer<T> {
    which: Cell<bool>,
    // we use UnsafeCell here because we are formally sharing the data, even if they are all accesses are disjoint
    data: [UnsafeCell<T>; 2],
}

pub struct WriteHandle<T> {
    buffer: Rc<DoubleBuffer<T>>,
}

pub struct ReadHandle<T> {
    buffer: Rc<DoubleBuffer<T>>,
}

impl<T> Clone for ReadHandle<T> {
    // ... the obvious impl ...
}
```

Why is this path so much better? Because now that the read and write halves of our use-case are now split, we can optimize them separately.
This goes back to point 1. And in doing so, we are also managing the lifetime ourselves in the library, so users aren't able to mess up.

Nice! Let's get the basic API up and running then.

```rust
impl<T> WriteHandle<T> {
    pub fn new(buffers: [T; 2]) -> Self {
        Self {
            buffer: Rc::new(DoubleBuffer {
                which: Cell::new(false),
                data: buffers.map(UnsafeCell::new),
            })
        }
    }

    pub fn swap_buffers(&mut self) {
        let which = self.buffer.which.get();
        self.buffer.which.set(!which);
    }

    pub fn read_handle(&self) -> ReadHandle<T> {
        ReadHandle { buffer: self.buffer.clone() }
    }

    pub fn write_buffer(&mut self) -> &mut T {
        let which = self.buffer.which.get();
        let buffer = &self.buffer.data[which as usize];
        // SAFETY: we only get exclusive access to `which`, and the read handle
        // only accesses `!which` these are disjoint, so they can't alias
        unsafe { &mut *buffer.get() }
    }

    pub fn read_buffer(&self) -> &T {
        let which = self.buffer.which.get();
        // SAFETY: we only get read access to `!which`, this is disjoint with
        // `which` so this can't alias WriteHandle::write_buffer
        let buffer = &self.buffer.data[(!which) as usize];
        unsafe { &*buffer.get() }
    }
}

impl<T> ReadHandle<T> {
    fn read_buffer(&self) -> &T {
        let which = self.buffer.which.get();
        // SAFETY: we only get read access to `!which`, this is disjoint with
        // `which` so this can't alias WriteHandle::write_buffer
        let buffer = &self.buffer.data[(!which) as usize];
        unsafe { &*buffer.get() }
    }
}
```

This API allows us to swap buffers, and easily access each buffer. The `ReadHandle`
can only access the current read buffer, and the `WriteHandle` can only mutably access the
current write buffer.

Alright, this looks like a good api. Do you see anything wrong with it? 


---

# Spoilers

It's unsound! The safety doc on `ReadHandle::read_buffer` is wrong, they can alias because of `WriteHandle::swap_buffers`

```rust
let mut writer = WriteHandle::new([0, 1]);
let reader = writer.read_handle();

let a = reader.read_buffer();
writer.swap_buffers();
let b = writer.write_buffer();
assert_eq!(*a, *b); // YIKES! this shouldn't be possible
```

Oh no. **OH NO**.

How do we fix this?

The problem is that the writer doesn't know that the reader already has access to the read buffer, so swapping buffers isn't safe.
So it looks like we will need to track how many readers are currently reading the buffer.

```rust
struct DoubleBuffer<T> {
    which: Cell<bool>,
    active_readers: Cell<u32>, /// <<--- new
    // we use UnsafeCell here because we are formally sharing the data, even if they are all accesses are disjoint
    data: [UnsafeCell<T>; 2],
}

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

impl<T> Drop for ReadGuard<'_, T> {
    fn drop(&mut self) {
        let readers = self.buffer.active_readers.get();
        self.buffer.active_readers.set(readers - 1);
    }
}

impl<T> ReadHandle<T> {
    pub fn read_buffer(&self) -> ReadGuard<'_, T> {
        let which = self.buffer.which.get();
        let readers = self.buffer.active_readers.get();
        self.buffer.active_readers.set(readers.checked_add(1).expect("Tried to read too many times"));
        ReadGuard {
            which,
            buffer: &self.buffer,
        }
    }
}

impl WriteHandle<T> {
    // -- snip --

    pub fn try_swap_buffers(&mut self) -> bool {
        if self.buffer.active_readers.get() == 0 {
            let which = self.buffer.which.get();
            self.buffer.which.set(!which);
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
```

Now if we try this we get a panic instead, which is not ideal, but the trade-off is better this time.
You only need to ensure that there are no active readers before swaps, but writes and reads can still
happen concurrently.

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

The key innovation here is ensuring that the writers cannot swap the buffers while there is an active reader.
This ensures the write buffer is not aliased, so we are allowed to get a `&mut _` to the write buffer.
This has the knock on effect that we cannot expose references from the `ReadHandle` because the `WriteHandle`
needs to know when the `ReadHandle` is accessing the read buffer to prevent bad swaps.

Now we have a workable API for double buffering. But it required us to dip into unsafe code.
Next time we'll figure out what this means and why it's ok in [Interlude on Unsafe](Double-Buffer-4.html).
