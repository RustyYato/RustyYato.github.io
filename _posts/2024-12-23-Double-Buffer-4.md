---
layout: post
title:  "Double Buffers - Interlude on Unsafe"
date:   2024-12-23 12:00:00 -0000
categories: unsafe,advanced rust,library design
---

What is `unsafe` in Rust, why does it exists, and how does it interact with the rest of the language.

This is a part of a series of blogs, you can see the full list [here](Double-Buffer-1.html).

`unsafe` in Rust is a formal notation for specifying when the compiler is unable to
automatically check the soundness of your program. This includes exactly these actions
specified by the [Rustonomicon](https://doc.rust-lang.org/nomicon/what-unsafe-does.html):

* Dereference raw pointers
* Call unsafe functions (including C functions, compiler intrinsics, and the raw allocator)
* Implement unsafe traits
* Access or modify mutable statics
* Access fields of unions

Why do these actions prevent the compiler from checking soundness, what even is soundness?

## Soundness and Undefined Behavior

[Soundness](https://en.wikipedia.org/wiki/Soundness) is a term Rust has borrowed from formal logic, it means

> Soundness := Assuming all unsafe code in you dependencies are sound, there is no way for safe code or unsafe code
> following any safety preconditions of your unsafe API to cause Undefined Behavior.

Now that's a mouthful. But isn't this circular with this condition? "... dependencies are sound ..."
Not quite, this definition is defined inductively. This only works because Rust enforces that the crate
dependency graph forms a [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph). But we've traded one
term for another, what is Undefined Behavior?

> Undefined Behavior := ...

I actually cannot accurately define this yet. Rust doesn't yet have a formal memory model. But it is starting
to grow one. You can direct questions about what is considered undefined behavior to [Zulip](https://rust-lang.zulipchat.com/#narrow/channel/136281-t-opsem)
or scour [unsafe-code-guidelines](https://github.com/rust-lang/unsafe-code-guidelines/) on Github.

That said, there is quite a lot that we can say about Rust's future memory model. For example, we know that the following are
all undefined behavior:

* data races[^1]
* invalid value representations of types (i.e. `2` is not a valid `bool`, but `0` and `1` are)
* dereferencing an unaligned, null, or dangling pointer
* and [more](https://doc.rust-lang.org/nomicon/what-unsafe-does.html)

Why does Rust have Undefined Behavior? For two reasons:
 1. to enable optimizations to make your program smaller, more memory efficient, or faster
    * for example by restricting value representations of types, you don't need a default branch
      in a `match` to maintain safety (like you would in C), and you get niche filling optimizations
      which reduce the size of values by exploiting these restrictions. i.e. `Option<bool>` is 1 byte large
 2. because systems that rust depends on also have Undefined Behavior, for example [LLVM](https://llvm.org/docs/UndefinedBehavior.html),
    your OS, and sometimes even your hardware (although this last one may just be a relic of the past now).
    For example, in LLVM doing a branch on uninitialized value is UB.

So soundness is an all-encompassing property of your API. It doesn't matter if "reasonable" programs don't exploit it,
because in Rust we want to allow safe code to fearlessly optimize as much as possible (via the compiler or the author).
Sound APIs are the gold standard, because they allow this.

## Safety Invariants

Some types may have invariants that their values must conform to. This means that they have some values that are considered unsafe,
even if they don't require unsafe to construct in the same module. We will call any values that conform to the invariant safe values.
Let's take `Vec`, which is something like this[^2]: 

```rust
struct Vec<T> {
    ptr: NonNull<T>,
    len: usize,
    capacity: usize
}
```

One part of the invariant is the `len` field should never be larger than the `capacity` field. 
But in the `std::vec` module, it is possible to create a `Vec` `Vec { ptr: NonNull::dangling::<T>(), len: 1, capacity: 0 }`.
So we say that `Vec` has a safety invariant that `len <= capacity`.

These are used to reason about the soundness of all APIS. Every function in the API must ensure that functions which take safe values must ensure that
those values are safe after the function is complete. For example, inside `std::vec` you must ensure that any function taking a `&mut Vec<T>` leaves
`len <= capacity` after the function is complete.

We can use invariants like this to ensure that some functions are safe. For example, `Vec::pop`

```rust
impl<T> Vec<T> {
    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            None
        } else {
            unsafe {
                self.len -= 1;
                /// here we know that self.len doesn't wrap around, and since is started with len <= capacity
                // and len - 1 < len <= capacity -> len - 1 <= capacity. So the invariant is preserved.

                Some(ptr::read(self.as_ptr().add(self.len())))
            }
        }
    }
}
```

This allows us to verify that `Vec::pop` is correct by the logic shown in the comment. And since we have that proof, we know that this function
is sound![^3]

## Unsafe Functions

Unsafe functions are functions where not all safe values are valid inputs. For example, take [`Vec::set_len`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.set_len)

```rust
impl<T> Vec<T> {
    unsafe fn set_len(&mut self, new_len: usize) { ... }
}
```

`usize::MAX` is a perfectly good value, but it is pretty much always incorrect as an input to `Vec::set_len`.

In fact here's the [safety precondition](https://doc.rust-lang.org/std/vec/struct.Vec.html#safety-4) for `Vec::set_len`
> * `new_len` must be less than or equal to `capacity()`.
> * The elements at `old_len..new_len` must be initialized.

There have two restrictions on what are considered valid inputs for `Vec::set_len`. So this function must be marked `unsafe` (and it is).

This reasoning also works for unsafe trait methods. For example, take [`GlobalAlloc::alloc`](https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html#tymethod.alloc).
It requires that the layout you pass to it has a non-zero size.

But who decides what these safety conditions are? For inherent methods and free functions, this is clear: the person who wrote the method/function. But for
trait methods this isn't so clear at first, is it the person who defined the trait or the implementor of the trait?
The current consensus is the person who defined the trait trait specifies the safety preconditions for any unsafe methods/associated functions,
and implementors may relax those requirements in some situations.

Note that a safe trait may have an unsafe function/method. This means that in a generic context, you must uphold the preconditions of that function, but
you don't get any guarantees about this function beyond it is safe to call if you uphold the preconditions. That seems like a bad bargain, which is why
this is very uncommon. Normally you will only see unsafe functions in unsafe traits.

## Robust Functions

This is a relatively new term, it describes a function which is allowed to take in unsafe values (as in values that don't satisfy their safety invariants).
This could be like a function that takes a formally `&str` but allowed any non-UTF8 strings. These are nice because they make it easier to use by unsafe code.

## Unsafe Traits

An unsafe trait is one where the implementor is required to uphold some guarantees. For example, [`GlobalAlloc`](https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html)
requires it's implementors of return a valid pointer or null from [`GlobalAlloc::alloc`] and [`GlobalAlloc::realloc`]. Any safe trait allows for arbitrary (and even malicious) implementations.

If you use a safe trait to constrain a generic type, you are only guaranteed that if you put safe values in, you get safe values out, nothing more, nothing less.
The safety of your code may not depend on anything beyond that. If you use a safe trait on a specific concrete type, you may get more guarantees depending on the type.
For example, `impl Ord for i32` does the obvious thing, and you may rely on it.

For unsafe traits, you may rely on the guarantees on the trait even if you are using a generic type.
But the major downside is now the trait is unsafe, so less people are willing to implement it. Sometimes this is a worthy tradeoff.

## Unsafe Blocks

An unsafe block is a unit of code that has a precondition. These are justified by appealing to safety invariants, safety preconditions (if inside an unsafe function), or
control flow before the block.

If you cannot justify an unsafe block via these two ways, then you likely have an unsound unsafe block. Now this may be fine if you control all usages of this block, for
example, for convenience you may wrap an unsafe block in a "safe" functions, and this is unsound. But if you control all call-sites of then you can justify the unsafe
block by appealing to that control. I would encourage you to stay away from this practice, since it makes code harder to maintain in the long run.

For example, this is a safe function which clears the vector with the safety justification documented.

```rust
fn clear_vec_i32(v: &mut Vec<i32>) {
    // SAFETY: 0 is less or equal to all usize (including v.capacity()) and
    // v.len()..0 is an empty range, so it is vacuously true that all elements
    // in that range are initialized
    unsafe { v.set_len(0) }
}
```

Now if we changed `i32` to any other type, this function would *remain safe*, but it may have the undesired behavior of leaking values (which isn't a problem for `i32`).
But since leak freedom isn't a guarantee that Rust makes, this is still considered safe.

---

### Footnotes

[^1]: Data races are a specific kind of race condition, and no turing complete language with concurrency can hope to put a lid on general race conditions.
    Because this is also a race condition:
    ```rust
    static DATA: Mutex<i32> = Mutex::new(0);
    /// in thread 1
    *DATA.lock().unwrap() = 1;
    // in thread 2
    *DATA.lock().unwrap() = 2;
    // in thread 3
    assert!(matches!(*DATA.lock().unwrap(), 0 | 1 | 2));
    ```
    There is nothing wrong with this code, but the value passed to the assert depends on which order the threads are run in. And this makes it a race condition.

    Data races are specifically when a a write to some memory location races with a read/write to the same memory location.
    The above example with the mutex doesn't have a data race exactly because we used a mutex to ensure that only one thread
    can write to the memory location at a given time.

    see the [Rustonomicon](https://doc.rust-lang.org/nomicon/races.html) for more details

[^2]: It is more complex than this in `std` because of many optimizations, and how important performance is for `Vec`, but these two representations are equivalent
      for `Vec<T>` (using the global allocator)

[^3]: Well, not quite, we need to consider the full safety invariant of `Vec`, which includes mentioning initialized elements, but that's omitted for brevity.
      Safe to say, this has been considered for the official implementation of `Vec`.