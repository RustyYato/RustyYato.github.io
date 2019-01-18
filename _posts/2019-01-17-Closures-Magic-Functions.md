---
layout: post
title:  "Closures: Magic Functions"
date:   2019-01-17 12:00:00 -0700
categories: rust syntactic sugar
---

If you want to respond to this post, please respond via [Rust users](https://users.rust-lang.org/t/blog-closures-magic-functions/24306) or [Reddit](https://www.reddit.com/r/rust/comments/ah307y/blog_closures_magic_functions/).


---

Closures seem like magical functions. They can do magic like capture their enviornment, which normal functions can't do.

```rust
let hello_world = "Hello World!".to_string();

let closure = || println!("{}", hello_world);

// error[E0434]: can't capture dynamic environment in a fn item
// fn normal_function() {
//     println!("{}", hello_world)
// }
```

How does this work?

The first thing to understand about closures is that they are pure sugar, and three traits working in concert.

## Fn* Traits

The three traits are
```rust
trait FnOnce<Args> { type Output; fn call_once(self, args: Args) -> Self::Output; }
trait FnMut<Args> : FnOnce<Args> { fn call_mut(&mut self, args: Args) -> Self::Output; }
trait Fn<Args> : FnMut<Args> { fn call(&self, args: Args) -> Self::Output; }
```
*note* I have removed some unnecessary details, like function calling convention to simplify. You can see the docs for more info about each one here: [FnOnce](https://doc.rust-lang.org/std/ops/trait.FnOnce.html), [FnMut](https://doc.rust-lang.org/std/ops/trait.FnMut.html), [Fn](https://doc.rust-lang.org/std/ops/trait.Fn.html).

These traits are critical to how closures work, so let's delve into how they work.

### Args

First the type parameter: `Args`

`Args` must always be a tuple representing the arguments of the closure. 
for example

```rust
|| "hi";               // this has Args = ()
|a: u32| ();           // this has Args = (u32,)
|a: f32, q: String| a; // this has Args = (f32, String)
```

This is to get around needing varadict generics to handle every possible list of arguments.
This representation is unstable, and may change in the future. So instead of using the `Fn*` traits
directly, you can use them like so

```rust
Fn() -> u32
FnMut(u32, u32)
FnOnce(String) -> Vec<u8>
```

### Output

Next is type associated type `Output`.

This is quite simple, it just represents the output type of the closure.

```rust
|| "hi";               // this has Output = &'static str
|a: u32| ();           // this has Output = ()
|a: f32, q: String| a; // this has Output = f32
```

### call*

Finally is the functions

`fn call*(self, args: Args) -> Self::Output;`

These functions do the leg work of executing the closure. There are a few notable differences between each one.

* In `FnOnce` we have `call_once`, which takes a `self` reciever. This is how it enforces that it is only called once. After self is moved into this function call, it can't be used again.
* In `FnMut` we have `call_mut`, which takes a `&mut self` reciever. This allows changes to the enviornment in closures.
* In `Fn` we have `call`, which takes a `&self` reciever. This doesn't allow chagnes to the enviornment (ignoring shared mutablility), but it does make it the most flexible type of closure. It can be called as many times as you want, and it can be thread-safe if `Send` and `Sync` are also implemented for the closure.

## Examples

I believe that working by example is the best way to explain something.

I will show the desugaring of a few closures, and explain why they are that way, and some benefits and costs to each closure.

Note: I will not show how `Send` and `Sync` are impled, as that is out of scope. After the first desugaring, I will not show the impls for all three `Fn*` traits, only the most specific one. So if you see `Fn`, then assume `FnMut` and `FnOnce` are impled with the same function body. If you see `FnMut`, then assume that `FnOnce` is impled with the same function body, but `Fn` is *not* impled. If you see `FnOnce`, then assume that `Fn` and `FnMut `are *not* impled. I will also put `type Output` in a comment to show what it would be if I only impl `Fn` or `FnMut`.

---
First, the simplest closure, one that doesn't capture anything, and only returns a unit.
```rust
let x = || ();
let y = x();
```
gets desugarred to
```rust
#[derive(Clone, Copy)]
struct __closure_0__ {}

impl FnOnce<()> for __closure_0__ {
    type Output = ();
    
    fn call_once(self, args: ()) -> () {
        ()
    }
}

impl FnMut<()> for __closure_0__ {
    fn call_mut(&mut self, args: ()) -> () {
        ()
    }
}

impl Fn<()> for __closure_0__ {
    fn call(&self, (): ()) -> () {
        ()
    }
}

let x = __closure_0__ {};
let y = Fn::call(&x, ());
```

[Playground Link __closure_0__](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=d43d7b0894c41e78365413a56bcfb20d)

Now, there is quite a bit to unpack here. First we get this new type `__closure_0__`. We can also see that `Clone` and `Copy` are derived for `__closure_0__`. This is because it is an empty type so it is trivial to `Clone` and `Copy` an empty struct. This allows for more flexiblity when using the closure.

Rust will pick the most specific `Fn*` trait to use whenever you call a function, in this order: `Fn`, `FnMut`, `FnOnce`. So in this case, because we can implement `Fn`, we implement that and all pre-requisites (`FnMut` and `FnOnce`). The function body from the closure is copied over to the function body of each of `call*` functions.

Then create the closure by creating this new struct. We call the closure by calling the most specific `call*` function, which in this case is `call`.

*Note:* the names I give, `__closure_0__` are arbitrary and the names that are actually used are random. This makes closures unnameable.

*Note:* How Rust knows which `Fn*` trait to derive for the closure is up to analysis of what it captures and how it is used (seen later).

You can use these playground links to test out the desugared code!

---
Now one step up, lets capture a variable.
```rust
let message = "Hello World!".to_string();
let print_me = || println!("{}", message);

print_me();
```
desugars to
```rust
#[derive(Clone, Copy)]
struct __closure_1__<'a> { // note: lifetime parameter
    message: &'a String, // note: &String, not String
}

impl<'a> Fn<()> for __closure_1__<'a> {
    // type Output = ();
    
    fn call(&self, (): ()) -> () {
        println!("{}", *self.message)
    }
}

let message = "Hello World!".to_string();
let print_me = __closure_1__ { message: &message };


Fn::call(&print_me, ());
```

[Playground Link __closure_1__](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=c6e017ceb489566268b1f0fd28f8be04)

We now have a field on `__closure_1__`, this represents the enviornment that is being used. So when we go to implement the `Fn*` traits, we use these fields to get access to the enviornment. Whenever Rust accesses one of these fields, it first dereferences them, the reason why will become evident when we get to mutating closures.

Notice the lifetime parameter on `__closure_1__`, because it is borrowing from the stack frame with `&message`, `print_me` has a non-`'static` lifetime. One downside to this is that it can't be sent across threads! Threads require a `'static` lifetime so that things don't deallocate while they run.

We still maintain `Clone` and `Copy` because shared references are `Copy`.

---
Now, what about if I have a closure with arguments? What about `move` closures?

```rust
let header = "PrintMe: ".to_string();
let print_me = move |message| println!("{}{}", header, message);

print_me("Hello World!");
```
desugars to
```rust
#[derive(Clone)]
struct __closure_2__ { // note: no lifetime parameter
    header: String     // note: String, not &String
}

impl<'a> Fn<(&'a str,)> for __closure_2__ {
    // type Output = ();
    fn call(&self, (message,): (&'a str,)) -> () {
        println!("{}{}", self.header, message);
    }
}

let header = "PrintMe: ".to_string();
let print_me = __closure_2__ {
     header: header // note: no &
};

Fn::call(&print_me, ("Hello World!",));
```

[Playground Link __closure_2__](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=4ebcba27e4163605964645b1f553c5f2)

First, the types of the closure arguments are resolved via type inference.

Next, how are arguments handled? As we saw earlier in the `Fn* Traits` section, arguments are really just a single tuple containing all of the arguments. This tuple is automatically created whenever we call a closure and destructured inside the `call*` function.

Finally, what did `move` do? Simply, instead of borrowing from the enviornment, we are going to move everything from the enviornment into this new anonomous struct (`__closure_2__`). Now because `__closure_2__` doesn't contain any lifetimes, it has a `'static` lifetime, which is necessary for it to be sent across threads! But in doing so, we also lost `Copy`, now our closure in only `Clone`. :(

This is why when you do anything with threads, you need to use `move` closures. They eliminate many of the references that would otherwise be created.

---
More on `move`

```rust
let a = "Hello World".to_string();
let a_ref = &a;

let print_me = move || println!("{}", a_ref);

print_me();
```
desugars to

```rust
// lifetimes are back, even though this is a `move` closure
// because this closure captures a reference
// note: a new lifetime parameter will be created for
// user-defined structs that also have lifetime parameters.
#[derive(Clone, Copy)]
struct __closure_3__<'a> {
    a_ref: &'a String
}

impl<'a> Fn<()> for __closure_3__<'a> {
    // type Output = ();

    fn call(&self, (): ()) {
        println!("{}", self.a_ref)
    }
}

let a = "Hello World".to_string();
let a_ref = &a;

// because this is a move closure, there are no new references here
let print_me = __closure_3__ { a_ref: a_ref };

Fn::call(&print_me, ());
```

[Playground Link __closure_3__](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=bdb0e9a398b07e3bead33b2ddf952edf)

Notice that even though we have a `move` closure, we still get lifetimes. This is because we have a reference from the enviornment. This means that unless that reference resolves to be `'static`, you cannot send it across threads. In this case the reference is definitely a shorter lifetime than `'static` 

---
What about returning things from closures, and mutating the enviornment inside a closure.

```rust
let mut counter: u32 = 0;
let delta: u32 = 2;

let next = || {
    counter += delta;
    counter
};

assert_eq!(next(), 2);
assert_eq!(next(), 4);
assert_eq!(next(), 6);
```
desugars to

```rust
struct __closure_4__<'a, 'b> {
    counter: &'a mut u32,
    delta: &'b u32
}

impl<'a, 'b> FnMut<()> for __closure_4__<'a, 'b> {
    // type Output = u32;

    fn call_mut(&mut self, (): ()) -> u32 {
        *self.counter += *self.delta;
        *self.counter
    }
}

let mut counter: u32 = 0;
let delta: u32 = 2;

let mut next = __closure_4__ {
    counter: &mut counter,
    delta: &delta
};

assert_eq!(FnMut::call_mut(&mut next, ()), 2);
assert_eq!(FnMut::call_mut(&mut next, ()), 4);
assert_eq!(FnMut::call_mut(&mut next, ()), 6);
```

[Playground Link __closure_4__](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=0fa923cb58ee9077b7788b37aa0ff372)

Because we are changing counter inside of the closure, we can implement at most `FnMut`. This is because we don't have write access inside of `Fn`.

We take a `&mut` to `counter` so that we can change it, and a `&` to delta to read from it. Each reference gets a fresh lifetime parameter.

We can now see why we need to dereference the references inside of `call*`. This is because we need the correct types for things to work out. For example, there is no impl of `AddAssign` for `&mut u32`, but there is one for `u32`. So we need to dereference `self.counter` so that Rust can resolve `AddAssign` correctly. There is nothing special about `AddAssign`,  type inference requires that these types are dereferenced in order to work correctly.

---
What about consuming things in a closure

```rust
let a = vec![0, 1, 2, 3, 4, 5, 100];

// notice, no `move`
let transform = || {
    let a = a.into_iter().map(|x| x * 3 + 1);
    a.sum::<u32>()
};

println!("{}", transform());
// println!("{}", transform()); // error[E0382]: use of moved value: `transform`
```
desugars to

```rust
#[derive(Clone)]
struct __closure_5__ {
    a: Vec<u32> 
}

impl FnOnce<()> for __closure_5__ {
    type Output = u32;
    
    fn call_once(self, (): ()) -> u32 {
        let a = self.a.into_iter().map(|x| x * 3 + 1);
        a.sum::<u32>()
    }
}

let a = vec![0, 1, 2, 3, 4, 5, 100];

let transform = __closure_5__ { a: a };

println!("{}", FnOnce::call_once(transform, ()));
// println!("{}", transform.call_once(())); // error[E0382]: use of moved value: `transform`
```

[Playground Link __closure_5__](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=ca812f04fd250c760c5fbc328d9590fe)

Even though we didn't add the `move` qualifier to the closure we see that `a` was moved into the closure. This is because `Vec::into_iter` takes `self` be value, which means `self` will be moved into the function. Because of this, the Rust moves a into `__closure_5__`. This means that `a` must be consumed during the function call, only `FnOnce` can be implemented.
