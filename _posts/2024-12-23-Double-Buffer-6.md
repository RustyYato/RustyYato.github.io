---
layout: post
title:  "Double Buffers - Interlude on Atomics"
date:   2024-12-23 12:00:00 -0000
categories: unsafe,advanced rust,library design
---

A crash course on atomics!

This is a part of a series of blogs, you can see the full list [here](Double-Buffer-1.html).

[Last time](Double-Buffer-5.html) we implemented a sound double buffer, and we had to dip into
atomics to get a similar implementation to our single-threaded implementation. I was a bit hand-wavey
about it, so now it's time to dig into atomics and how to work with them.

This will be a crash course, I will only be explaining just enough to get going. If you need a more
in depth look at atomics, how they are implemented in hardware, and a more in-depth explanation on
how to use them, check out Mara Bos's [Rust Atomics and Locks](https://marabos.nl/atomics/) book!
(It's available online for free, but I highly recommend you buy it to support their work!).
If you do end up reading that, you can safely skip this section and move onto the [next one](Double-Buffer-7.html), where
we'll get our hands dirty with a production grade implementation of multi-threaded double buffers.

## What is atomic?

In the context of multi-threading or larger scale synchronization, at operation is atomic when
you can only see the effects of the operation before it was done, or after it was done, but not
while the operation is happening. For example, a database transaction is atomic, because
you cannot see any writes that were made inside the transaction from outside the transaction
until it is completed.

An atomic operation is a very useful concept, since we can rely on no one else seeing the
intermediate state. This reduces the proof burden on us to show that an algorithm is correct.
Note here that, an atomic operation is a more general concept than hardware atomics (which in Rust
exposed as special types). But in the context of multi-threading, these are often conflated.
So you will often see people talking about "atomics", and they will mean hardware atomics.
I will try to be clear about which meaning I am talking about, and it should be clear from context.

## What are hardware atomics?

Hardware Atomics are a set of low-level instructions exposed by most hardware that allow you to
communicate across cpu-cores without data-loss. They form the foundations of how all
multi-threaded synchronization primitives, data structures, and algorithms work.

In Rust, these are exposed as a set of types named like `Atomic*`, (i.e. `AtomicBool`, `AtomicUsize`, `AtomicPtr`, etc.).
Each of these types expose an API which takes a `&self` and mutates the shared value safely, by using specific hardware
instructions to synchronize the access.

Hardware atomics guarantee that writes and reads will not tear, and what side-effects you will
see if you read a value from an atomic. 

## Performance Classes

#TODO add a note about different performance classes

## How do you use atomics?

Let's take `AtomicUsize` for example, it has these functions (among others):

```rust
impl AtomicUsize {
    /// reads a value from the `AtomicUsize`
    fn load(&self, order: Ordering) -> usize { ... }
    /// writes a value into the `AtomicUsize`
    fn store(&self, value: usize, order: Ordering) { ... }
    /// adds `value` to the value in this `AtomicUsize`
    fn fetch_add(&self, value: usize, order: Ordering) -> usize { ... }
    /// we'll go over this in more details below
    fn compare_exchange(&self, value: usize, success: Ordering, failure: Ordering) -> Result<usize, usize> { ... }
}
```

Each of these functions takes at least 1 an `Ordering` argument. The `Ordering`
specifies which side-effects the current thead can see. This is specified by
a *happens before* relation at two points in a per-thread clock and a *synchronizes with*
relation on two operations. In order to
specify this relation, we need to take a quick peek at `Ordering`:

```rust
pub enum Ordering {
    Relaxed,
    Release,
    Acquire,
    AcqRel,
    SeqCst,
}
```

Now, let's specify *happens before*

Within a given thread, if operation A which was executed before to an operation B,
then operation A *happens before* operation B.

Between two different threads given
 1. an operation A, which *happens before* an operation X
 2. an operation Y, which *happens before* an operation B
 3. operation X *synchronizes with* operation Y
then operation A *happens before* operation B

To reduce verbosity, I will use the following notation
`Release+` = `Release`, `AcqRel`, or `SeqCst`
`Acquire+` = `Acquire`, `AcqRel`, or `SeqCst`

Where you can read `Release+` as, at least `Release` ordering and  `Acquire+` as, at least `Acquire` ordering

The given pairs of operations specify the *synchronizes with* relation
a store on variable X with `Release+` ordering and a load on variable X with `Acquire+` ordering,
*and* the load reads the value that was written by the store.

While this will be all we need to use atomics, it isn't fully complete. Rust takes this from C++20, so see
[their docs](https://en.cppreference.com/w/cpp/atomic/memory_order) for more details.

Now, let's work through some examples to check our understanding.

### Example 1

For example, let's take this example.
```rust
static A: AtomicUsize = AtomicUsize::new(0);
static B: AtomicUsize = AtomicUsize::new(0);

// in thread 1
let b = B.load(Ordering::Relaxed);
let a = A.load(Ordering::Relaxed);
println!("{a},{b}");

// in thread 2
A.store(1, Ordering::Relaxed);
B.store(1, Ordering::Relaxed);
```
which prints are allowed (select all that apply)
1. 0,0
2. 0,1
3. 1,0
4. 1,1

<details> <summary> Spoilers </summary>

All of them! Since we are using `Relaxed`, the only thing that's guaranteed is that
the wries won't tear. In particular, we don't get any guarantees about which store
we will see in thread 1. 

</details>

### Example 2

For example, let's take this example.
```rust
static A: AtomicUsize = AtomicUsize::new(0);
static B: AtomicUsize = AtomicUsize::new(0);

// in thread 1
let b = B.load(Ordering::Acquire);
let a = A.load(Ordering::Relaxed);
println!("{a},{b}");

// in thread 2
A.store(1, Ordering::Relaxed);
B.store(1, Ordering::Release);
```
which prints are allowed (select all that apply)
1. 0,0
2. 0,1
3. 1,0
4. 1,1

<details> <summary> Spoilers </summary>

Option 3 is impossible now, because if we see <code>b == 1</code> then we
are guaranteed to see the write to <code>A</code> because
<ol>
<li> the store to A is executed before the store to B in thread 2 </li>
<li> the load from B is executed before the load from A in thread 1 </li>
<li> the load from B in thread 1 <i>synchronizes with</i> the store to B in thread 2 if we read 1 </li>
</ol>

</details>

#TODO write more examples