---
layout: post
title:  "Double Buffers - Humble Beginnings"
date:   2024-12-23 12:00:00 -0000
categories: unsafe,advanced rust,library design
---

The basics of double buffering, requirements, and performance considerations.

This is a part of a series of blogs, you can see the full list [here](Double-Buffer-1.html).

In this section, we will go what double buffers are and how we are going to use them.

A simple naive double buffer is something like this:
```rust
struct DoubleBuffer<T> {
    which: bool,
    buffers: [T; 2],
}

impl<T> DoubleBuffer<T> {
    pub fn swap_buffers(&mut self) {
        self.which ^= true;
    }

    pub fn get_write_buffer(&mut self) -> &mut T {
        &mut self.buffers[self.which as usize]
    }

    pub fn get_read_buffer(&self) -> &T {
        &self.buffers[!self.which as usize]
    }
}
```

And while something like this may have worked in other languages, this is not so useful in Rust as-is
because we cannot use both the read buffer and writer buffer at the same time. Even though they are completely disjoint.
We could add a patch to this like so to work around this:

```rust
impl DoubleBuffer<T> {
    pub fn swap_buffers(&mut self) {
        self.which ^= true;
    }

    pub fn get_buffers(&mut self) -> (&mut T, &T) {
        let [a, b] = &mut self.buffers;
        match self.which {
            false => (a, b),
            true => (b, a),
        }
    }
}
```

But this is still difficult to use. Let's take a look at how this buffer could be used.


```rust
let mut buf = DoubleBuffer {
    which: false,
    // two 16x16 256-color images for example
    buffers: [vec![0; 16 * 16], vec![0; 16 * 16]],
};

let mut state = ...;

buf.swap_buffers();
let (writer, read) = buf.get_buffers();
print_buffer(read);
update(writer, &mut state);

buf.swap_buffers();
let (writer, read) = buf.get_buffers();
print_buffer(read);
update(writer, &mut state);
```

There are a few problems right now that we can immediately see:
 1. updates and reads are intrinsically tied together, so we can't scale read performance at all!
 2. every time we swap buffers we need to explicitly re-acquire the buffers, this is just Terrible UX
 3. managing the lifetime of the double buffer is tricky if you want to share it

Point 1 this is important because, we are eventually going to use this to implement a concurrent hash map,
and there read performance is very important. Since these maps tend to be read far more than they are written to.
So in order to benefit from that, we will need to ensure that we can optimize reads independently of writes.

Let's try to solve point 3 at least, since we're single-threaded anyways right now lets just use `Rc`/`RefCell`.
We'll figure out how to improve this later.

```rust
let mut buf = Rc::new(RefCell::new(DoubleBuffer {
    which: false,
    // two 16x16 256-color images for example
    buffers: [vec![0; 16 * 16], vec![0; 16 * 16]],
}));

let mut state = 0;

let printer = {
    let buf = buf.clone();

    move || {
        print_buffers(buf.borrow().get_read_buffer())
    }
};
let mut updater = {
    let buf = buf.clone();

    move || {
        update(buf.borrow_mut().get_write_buffer(), &mut state)
    }
};


// for use in swaps
let mut buf = buf.borrow_mut();

buf.swap_buffers();
printer();
updater();

buf.swap_buffers();
printer();
updater();
```

Nice, now the logic part is a bit simpler. Is this good enough? Let's try it!

```rust
thread 'main' panicked at src/main.rs:17:31:
already mutably borrowed: BorrowError
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Right..., we can't just factor out the `borrow_mut`. But this should work.

```rust
// -- snip --

// let mut buf = buf.borrow_mut();

buf.borrow_mut().swap_buffers();
printer();
updater();

buf.borrow_mut().swap_buffers();
printer();
updater();
```

But this is sad, Rust should have helped us catch this bug! But since we used
`RefCell` we opted into runtime borrow checking. Is there something we could
do to improve this?

Before moving on the the next part, [A Fresh Start](Double-Buffer-3.html), here are some questions
to think about and try to answer.

* How would you make this harder to misuse without making it too difficult to use?
* What other use-cases can you think of for double buffers?