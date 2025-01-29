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

When a reader's `count` is odd, then it is currently reading, if it is even it is inert.

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

First the data structures. We need a shared list for the epochs, and an atomic boolean flag
to keep track of which buffer is in use. We need to keep a pointer to the epoch
in the `ReadHandle` to avoid grabbing the lock in `DoubleBuffer` on every read.
In the `WriteHandle` we need to store some scratch space, which we will use to check
if a swap is complete, and amortize the cost of the allocation.
Then just make `WriteHandle` and `ReadHandle` thread-safe but using `Arc` instead of `Rc`.

We also need to ensure that `DoubleBuffer` is `Sync` under the right conditions.
From thread safety in [Interlude on Unsafe](Double-Buffer-4.html), we can justify
`Send` and `Sync` since
1. `DoubleBuffer<T>` doesn't have any unsynchronized shared ownership if `T` doesn't have unsynchronized shared ownership
2. `DoubleBuffer<T>` doesn't have any unsynchronized shared mutation if `T` doesn't have unsynchronized shared ownership
    * This isn't strictly required since we are using `Arc`, but if we instead used references, then 
        this does become required. So to be on the safe side we will require it.
        For references this is because the `WriteHandle` will hold a `&DoubleBuffer<T>`, and we need to ensure that
        this type is only `Send` if `T: Send`. Otherwise we would be able to send a `MutexGuard` to another thread.
        Which is unsound on some platforms.
3. `DoubleBuffer<T>` doesn't have any unsynchronized shared mutation if `T` doesn't have unsynchronized shared mutation

```rust
struct DoubleBuffer<T> {
    which: AtomicBool,
    epochs: Mutex<Vec<Arc<AtomicUsize>>>,
    cv: Condvar,
    data: [UnsafeCell<T>; 2],
}

pub struct WriteHandle<T> {
    last_epochs: Vec<usize>,
    buffer: Arc<DoubleBuffer<T>>,
}

pub struct ReadHandle<T> {
    epoch: Arc<AtomicUsize>,
    buffer: Arc<DoubleBuffer<T>>,
}

struct ReadGuard<'a, T> {
    which: bool,
    handle: &'a ReadHandle<T>,
}

// from &mut DoubleBuffer, we can access &mut T (via `Drop` for example)
// and we don't have shared ownership of T
// Mutex<Vec<...>> is Send
unsafe impl<T: Send> Send for DoubleBuffer<T> {}
// from &DoubleBuffer, we can access &mut T, and &T
// Mutex<Vec<...>> is Sync
unsafe impl<T: Send + Sync> Sync for DoubleBuffer<T> {}

impl<T> WriteHandle<T> {
    pub fn new(buffers: [T; 2]) -> Self {
        Self {
            last_epochs: Vec::new(),
            buffer: Arc::new(DoubleBuffer {
                which: AtomicBool::new(false),
                epochs: Mutex::new(Vec::new()),
                cv: Condvar::new(),
                data: buffers.map(UnsafeCell::new),
            })
        }
    }
}
```

Then to create new readers, we need to lock the `epochs` list, add a new element,
then store that in a `ReadHandle`.  This way each `epoch` is uniquely associated
with a single reader, and the `WriteHandle` can keep track of which readers are still
reading.

```rust
impl<T> DoubleBuffer<T> {
    fn create_reader(self: Arc<Self>) -> ReadHandle<T> {
        let epoch = Arc::new(AtomicUsize::new(0));
        self.epochs.lock().unwrap().push(epoch.clone());
        ReadHandle {
            epoch,
            buffer: self,
        }
    }
}

impl<T> WriteHandle<T> {
    pub fn read_handle(&self) -> ReadHandle<T> {
        self.buffer.clone().create_reader()
    }
}

impl<T> Clone for ReadHandle<T> {
    fn clone(&self) -> Self {
        self.buffer.clone().create_reader()
    }
}
```

Now let's implement the read algorithm described above. Note: we need to check that the epoch
is even before we update it to make sure that a `ReadGuard` wasn't leaked. Otherwise
a malicious user may leak a `ReadGuard`, start a new read, and then cause a data race
since the `WriteHandle` assumes that all even epochs are readers which aren't currently reading.

Then after completing the read, we need to notify the writer, which may be sleeping on the condvar
to ensure that it wakes up and continues the swap.

Note: we must use `&mut self` to prevent someone from starting a read while there is already a read in progress.
Since we are not handling reentrant reads. If someone wants to make multiple reads at a time, then they
should use multiple `ReadHandle`s.

```rust
impl<T> ReadHandle<T> {
    pub fn read(&mut self) -> ReadGuard<'_, T> {
        let epoch = self.epoch.load(Ordering::Relaxed);
        assert!(epoch % 2 == 0, "Read failed: detected a leaked `ReadGuard`, which put this `ReadHandle` into an invalid state");

        self.epoch.fetch_add(1, Ordering::AcqRel);
        let which = self.buffer.which.load(Ordering::Acquire);
        ReadGuard {
            which,
            handle: self,
        }
    }
}

impl<T> Drop for ReadGuard<'_, T> {
    fn drop(&mut self) {
        self.handle.epoch.fetch_add(1, Ordering::Release);
        self.handle.buffer.cv.notify_one();
    }
}
```

Onwards to the `WriteHandle`'s swap methods. We will be splitting this into three parts,
one for the setup phase, one for the iterative phase, and one combining them together.
First let's start the setup phase.

We need to garbage collect the dead readers, and setup the `last_epochs` buffer.

```rust
struct SwapInfo {
    start: usize,
}

impl<T> WriteHandle<T> {
    unsafe fn start_swap(&mut self) -> SwapInfo {
        let buffer = &*self.buffer;
        let mut epochs = buffer.epochs.lock().unwrap();

        self.last_epochs.clear();
        epochs.retain(|epoch| {
            if Arc::strong_count(epoch) == 1 {
                // if the reader is no longer with us, then we don't need to keep track
                // of this epoch anymore
                return false;
            }

            self.last_epochs.push(epoch.load(Ordering::Acquire));

            true
        });

        SwapInfo {
            start: 0,
        }
    }
}
```

Then to check if the swap is completed, we need to go through the readers
and check if they are done reading yet. We will only check readers
which were reading during setup, and we will skip over all readers
we have already verified are not reading the current buffer.

```rust
impl SwapInfo {
    fn is_finished(&mut self, epochs: &[Arc<AtomicUsize>], last_epochs: &[usize]) -> bool {
        let epochs = &epochs[self.start..];
        let last_epochs = &last_epochs[self.start..];

        for (i, (epoch, &last_epoch)) in epochs.iter().zip(last_epochs).enumerate() {
            if last_epoch % 2 == 0 {
                continue;
            }

            let now = epoch.load(Ordering::Acquire);

            if now == last_epoch {
                // if the reader is currently reading from the buffer
                // then set the start index to this epoch and return false
                self.start += i;
                return false;
            }
        }

        // if all readers are done reading from the current buffer
        // then set the start index to the end and return true

        self.start = last_epochs.len();

        true
    }
}
```

Finally, to finish the swap, we just loop until `SwapInfo::is_finished`
returns true, and wait on a `Condvar`, this way we aren't burning CPU time.

```rust
impl<T> WriteHandle<T> {
    unsafe fn finish_swap(&mut self, mut swap: SwapInfo) {
        let mut epochs = self.buffer.epochs.lock().unwrap();

        while !swap.is_finished(&epochs, &self.last_epochs) {
            epochs = self.buffer.cv.wait(epochs).unwrap();
        }
    }

    pub fn swap_buffers(&mut self) {
        let swap = unsafe { self.start_swap() };
        unsafe { self.finish_swap(swap) }
    }
}
```

We will see why we split this up next time in [next time](Double-Buffer-8.html).

Finally let's provide accessors. Note that it doesn't really matter which buffer is 
accessed, as long as readers can't access the mutable buffer that the writes have access to.
We will use `!self.which` for the reader buffer and `self.which` for the writer buffer.

```rust
impl<T> DoubleBuffer<T> {
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

impl<T> WriteHandle<T> {
    pub fn read_buffer(&self) -> &T {
        self.buffer.read_buffer()
    }

    pub fn write_buffer(&self) -> &T {
        // SAFETY: we are the writer
        unsafe { self.buffer.write_buffer() }
    }

    pub fn write_buffer(&mut self) -> &mut T {
        // SAFETY: we are the writer
        unsafe { self.buffer.write_buffer() }
    }
}

impl<T> Deref for ReadGuard<T> {
    type Target = T;

    fn deref(&self) -> &T {
        self.handle.buffer.read_buffer()
    }
}
```