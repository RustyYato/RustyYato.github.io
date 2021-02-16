---
layout: post
title:  "Generalizing over Generics in Rust (Part 1) - AKA Higher Kinded Types in Rust"
date:   2021-02-15 12:00:00 -0000
categories: type system,type families
---

Do you want to use Higher Kinded Types (`HKT`) in Rust? Are you tired of waiting for `GAT`? Well, this is the place to be.

It's easiest to understand `HKT` by analogy. In programming we have values. If you want to generalize over many different values, you use types. If you want to generalize over many different types, you use polymorphism (generics/templates/dynamic dispatch). And it normally stops there, but what if you want to generalize over the kinds of polymorphism? Specifically, you want to generalize over all things `T<U>`, where we're generic over `T`. How to do that? That's where `HKT` comes in.

Let's introduce some terminology. Values like `0`, `true`, and `x` in `x: bool` are called `term`s. To generalize over `value`s we use `type`s, like `bool` in `x: bool` or `Vec<i32>` in `vec![0, 1, 2]: Vec<i32>`. To generalize over `type`s  we use `kind`s, like `Type` (aka `*` in formal literature) or `Term`. In Rust we even have a third kind, `Lifetime`!

We can then reimagine generics are just functions in kinds. For example:
* `Vec` (Not `Vec<T>`, but `Vec`) is just `Type -> Type`
    * a function that takes a `Type` as argument, and produces a `Type`
* `Result` is `Type -> Type -> Type` (a function that takes two `Type` arguments and produces a `Type`)
* arrays are `Type -> usize -> Type`
* references are `Lifetime -> Type -> Type`.

With this notion of `kind`s we are now well equipped to generalize over generics. Just one cinch, we `Rust` doesn't know about `kind`s! So how do we generalize over generics if we can't even express the fundamental building blocks!

# Type Families

The key insight is that we can represent `kind`s as a system of types and traits. For example, here's how to handle `Type -> Type` kinds.

Note: I'll show how `Option` fits in, and introduce `Result<_, E>`, but I'll leave the implementation of `Vec` to you. It's an interesting puzzle if you are inclined

```rust
// `Option` has the kind `Type -> Type`
struct OptionFamily;
// `Result` has the kind `Type -> Type -> Type`,
// so we fill in one of the types with a concrete one
struct ResultFamily<E>(PhantomData<E>);

// I'll leave the implementation of `VecFamily` to you

// This trait represents the `kind` `Type -> Type`
pub trait Family<A> {
    // This represents the output of the function `Type -> Type`
    // for a specific argument `A`.
    type This;
}

impl<A> Family<A> for OptionFamily {
    // `OptionFamily` represents `Type -> Type`,
    // so filling in the first argument means
    // `Option<A>`
    type This = Option<A>;
}

impl<A, E> Family<A> for ResultFamily<E> {
    // note how all results in this family have `E` as the error type
    // This is similar to how currying works in functional languages
    type This = Result<A, E>;
}
```

Let's also introduce a type alias for ease of use. 

```rust
// Option<A> == This<OptionFamily, A>
pub type This<T, A> = <T as Family<A>>::This;
```

Now we can replace all usage of `Option`, `Result<_, E>` with `OptionFamily` or `ResultFamily<E>`! From this we can build up abstractions that "require" HKT. Let's start with the simplest abstraction `Functor`. This is just any `Type -> Type` that has a notion of mapping a value. Generally `Functor` represents a collection, but it can do much more than that (But I won't dive into `Functor` in particular in this post).

```rust
trait Functor<A, B>: Family<A> + Family<B> {
    fn map<F>(self, this: This<Self, A>, f: F) -> This<Self, B>
    where
        F: Fn(A) -> B + Copy;
}
```

This is quite a lot, so let's soak it in.

```rust
Family<A> + Family<B>
```

First, we require that our families can be used with either type (this allows you to restrict the families in some ways, for example for `HashSet`, only allow `T: Hash + Eq`). `A` will be the input type, and `B` will be the output type.

```rust
fn map<F>(self, this: This<Self, A>, f: F) -> This<Self, B>
where
    F: Fn(A) -> B + Copy;
```

Next we have this monstrous function! This will be easier to explain if we have a concrete family.

```rust
/// for `OptionFamily`
fn map<F>(self, this: Option<A>, f: F) -> Option<B>
where
    F: Fn(A) -> B + Copy;
```

So this is just the same as the ordinary map in `Option`, except it requires `Fn`, instead of `FnOnce`. We could generalize over `Fn` and `FnOnce` if Rust had associated traits, but alas we don't, so we'll have to make do with `Fn` (I'll leave it to you to figure out a nice way to abstract over `Fn`, `FnMut` and `FnOnce`). All we've done is generalize it to *any* family, not just `Option`.

The `Copy` bound is nice for things like `Vec` where you need to apply the function multiple times. You can convert any `Fn` to a `Fn + Copy` because `&_` implement `Fn` and are `Copy`.

So all together again:

```rust
trait Functor<A, B>: Family<A> + Family<B> {
    fn map<F>(self, this: This<Self, A>, f: F) -> This<Self, B>
    where
        F: Fn(A) -> B + Copy;
}
```

`Functor` just specifies a mapping operation. We can implement this rather easily.

```rust
trait<A, B> Functor<A, B> for OptionFamily {
    fn map<F>(self, this: This<Self, A>, f: F) -> This<Self, B>
    where
        F: Fn(A) -> B + Copy {
        // I'm not cheating!
        this.map(f)
    }
}

// try out `VecFamily`, it doesn't need to be optimal, it just needs to work!
```

Ok, but that's boring, I hear you say. What about `Monad`? Typically it's defined by it's `bind` operation (We'll skip `Applicative` here, try it our yourself!). I'm not going to explain how `Monad` works, or why you would want it, just how to implement it.

```rust
trait Monad<A, B>: Functor<A, B> {
    fn bind<F>(self, a: This<Self, A>, f: F) -> This<Self, B>
    where
        F: Fn(A) -> This<Self, B> + Copy;
}
```

Looks the same as `Functor` to me! ... Wait, the bound on `F` looks funky. Let's dig in to the `OptionFamily` implementation again, maybe that will clear things up.

```rust
/// for `OptionFamily`
fn bind<F>(self, a: Option<A>, f: F) -> Option<B>
where
    F: Fn(A) -> Option<B> + Copy;
```

This looks very familiar, ... where have I seen this before? `Option::and_then` is that you?

```rust
trait<A, B> Monad<A, B> for OptionFamily {
    fn bind<F>(self, this: This<Self, A>, f: F) -> This<Self, B>
    where
        F: Fn(A) -> This<Self, B> + Copy {
        // It fits 😉
        this.and_then(f)
    }
}

// try out `VecFamily`, it doesn't need to be optimal, it just needs to work!
```

In this way you can implement most, if not all `HKT` abstractions in stable Rust. However the ergonomics of these abstractions are downright abysmal. Bounds, bounds, bounds *everywhere*. We saw this a little in `Functor<A, B>`, we needed `Family<A> + Family<B>` (why does family need to be repeated!). Hopefully we can solve this on nightly, find out next time ... 

If you can't wait and must know more, check out [`type-families`](https://github.com/rustyyato/type-families), an experimental crate that implements the ideas outlined in this blog post.

You can discuss this on reddit [here](https://www.reddit.com/r/rust/comments/ll9un4/generalizing_over_generics_in_rust_part_1_aka/) or the users.rust-lang.org [here](https://users.rust-lang.org/t/generalizing-over-generics-in-rust-part-1-aka-higher-kinded-types-in-rust/55716/2)