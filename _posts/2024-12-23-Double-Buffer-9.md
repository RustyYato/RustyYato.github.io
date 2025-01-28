---
layout: post
title:  "Double Buffers - Concurrent HashMap"
date:   2024-12-23 12:00:00 -0000
categories: unsafe,advanced rust,library design
---

Let's use the double buffer to create a concurrent hash map!

This is a part of a series of blogs, you can see the full list [here](Double-Buffer-1.html).

[Previously](Double-Buffer-8.html) we implemented all the tools to make a concurrent hashmap.
So now let's get to it! We shouldn't need any unsafe code for now. All the tricky synchronization
unsafe-code is neatly wrapped up at lower levels of abstraction.

Note that everything we have gone over so far is going to be in the `buffers` module in the code examples.
So you will see `buffers::` and that is a reference to code we have already gone over.

First off, let's specify which operations we are going to implement.

```rust
enum HashMapOp<K, V> {
    Insert { key: K, value: V },
    Remove { key: K },
    Retain { func: Box<dyn FnMut(&K, &mut V) -> bool + Send + Sync> }
}

impl<K: Clone + Hash + Eq, V: Clone> buffers::Operation<HashMap<K, V>> for HashMapOp<K, V> {
    fn apply(&mut self, map: &mut HashMap<K, V>) {
        match self {
            Insert { key, value } => {
                map.insert(key, value);
            },
            Remove { key } => {
                map.remove(&key);
            },
            Retain { func } => map.retain(func),
        }
    }
}
```

In this blog, we will only handle these operations for simplicity, but you may handle more!
Now let's setup the basic handles we will use to interact with the map.

```rust
struct WriteHandle<K, V> {
    handle: buffers::OpWriteHandle<HashMap<K, V>, HashMapOp<K, V>>,
}

struct ReadHandle<K, V> {
    handle: buffers::ReaderHandle<HashMap<K, V>>
}

// a read guard represents a lock on the hashmap, and writers are not allowed
// to swap if there is an old enough read guard still alive
struct ReadGuard<'a, K, V> {
    guard: buffers::ReaderGuard<'a, HashMap<K, V>>
}
```

Now because we are using `OpWriteHandle` the `ReaderGuard` doesn't block on swaps immediately,
instead it will block the second swap after it was created. However, we expect that users will
not keep reader guards around too long, so this means we normally expect `publish` to be very fast.

```rust
impl<K, V> WriteHandle<K, V> {
    pub fn new() -> Self {
        Self {
            handle: buffers::OpWriteHandle::new(
                buffers::DelayWriteHandle::new(
                    buffers::WriteHandle::new([HashMap::new(), HashMap::new()])
                )
            )
        }
    }

    pub fn read_handle(&self) -> ReaderGuard {
        self.handle.get().read_handle()
    }
}

impl<K: Clone + Hash + Eq, V: Clone> WriteHandle<K, V> {
    pub fn insert(&mut self, key: K, value: V) {
        self.handle.push(HashMapOp::Insert { key, value })
    }

    pub fn remove(&mut self, key: K, value: V) {
        self.handle.push(HashMapOp::Remove { key, value })
    }

    pub fn retain<F: FnMut(&K, &mut V) + Send + Sync>(&mut self, f: F) {
        self.handle.push(HashMapOp::Retain { func: Box::new(f) })
    }

    pub fn publish(&mut self) {
        self.handle.swap_buffers()
    }
}
```

And for the reader is just a very thin wrapper around `buffers::ReadHandle`
and `buffers::ReadGuard`.

```rust
impl<K: Hash + Eq, V> ReadHandle<K, V> {
    pub fn read(&self) -> ReadGuard<'_, K, V> {
        ReadGuard { guard: self.handle.read() }
    }
}

impl<K, V> Deref for ReadGuard<'_, K, V> {
    type Target = HashMap<K, V>;

    fn deref(&self) -> &Self::Target {
        &self.guard
    }
}
```

With that we now have a complete eventually consistent concurrent hash map.
Wow! All that prep work make the hashmap portion pretty trivial. Next time,
we will work on generalizing the work we've done so far, in preparation
for a new and improved synchronization algorithm. 