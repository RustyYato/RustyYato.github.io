---
layout: post
title:  "Type Families Part 1 - AKA Higher Kinded Types in Rust"
date:   2021-02-11 12:00:00 -0700
categories: type system
---

Do you want to use Higher Kinded Types (`HKT`) in Rust? Are you tired of waiting for `GAT`? Well, this is the place to be.

It's easiest to understand `HKT` by analogy, in programming we deal with values, these values have types. If you want to generalize over many different types of values, you use generics (or equivilent). But what if you want to generalize even further? What does that even mean?

One way to generalize further is to generalize over type constructor. What's a type constructor? It's basically a function that makes a type. For example, in Rust `Vec` is a type constructor, it's a function from `Type -> Type`. If you apply a `i32` to it, then it's a `Vec<i32>` which is a `Type`. Of course in stable Rust you can't actually do anything with `Vec`, you have to specify a concrete type (`Vec<T>` is a concrete type, even if `T` is a generic paramter).

Let's dig a bit deeper. First let's setup some terminology, the actual name for these "functions" is `kind`. Let's list some `kind`s to get a feel for it:

- `Vec` as we saw before is `Type -> Type`
    - in standard notation `* -> *`, where `*` means `Type`, but I will not be using standard notation, because Rust throws a wrench into it in just a little bit!
- `Option<i32>` is a `Type`
- `Result` is a `(Type, Type) -> Type`, or equivilently `Type -> Type -> Type`

Rust adds in the extra complication of having not one, not two, but three different basic kinds:

- `Type` - these are your normal types, like `i32`, `Vec<i32>`, or `()`
- `Lifetime` - these are lifetime parameters (`'a`), `'static`, or the inferred lifetimes
- `Term` - These are `const-generic` parameters, `const X: usize` is a `Term`

This means that `Rust` can have a whole host of different kinds, like `Lifetime -> Type -> Type` (references) or `Type -> Term -> Type` (arrays).

So what if you wanted to generalize over type constructors, for example over all kinds `Type -> Type` (like `Vec` or `Option`), how would you do that? In languages that support `HKT`, like Haskell, it's as simple as

```haskell
functor_map :: (a -> b) -> m a -> m b
functor_map = fmap
```

But stable Rust doesn't support `HKT`, then how do we fullfil the promise at the top? And how does `GAT` fit in?

# Stable Type Families

The key insight is that we can represent `Vec`, `Option`, and `Result` as types, and kinds as traits. Here, I will only specify `Type -> Type` kinds, but this approach can be generalized to all kinds.

```rust
// `Option` has the kind `Type -> Type`
struct OptionFamily;
// `Result` has the kind `Type -> Type -> Type`,
// so we fill in one of the types with a concrete one
struct ResultFamily<E>(PhantomData<E>);

// I'll leave the implementation of `VecFamily` to you

pub trait Family<A> {
    type This;
}

impl<A> Family<A> for OptionFamily {
    type This = Option<A>;
}

impl<A, E> Family<A> for ResultFamily<E> {
    // note how all results in this family have `E` as the error type 
    type This = Result<A, E>;
}
```

Let's also introduce a type alias for ease of use. 

```rust
// Option<A> == This<OptionFamily, A>
pub type This<T, A> = <T as Family<A>>::This;
```

Now we can replace all usage of `Option`, `Result<_, E>` with `OptionFamily` or `ResultFamily<E>`! From this we can build up abstractions that "require" HKT. Let's start with the simplest abstraction `Functor`. This is just any `Type -> Type` that has a notion of mapping a value. Generally `Functor` represents a collection, but it can do much more than that.

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

First, we require that our familes can be used with either type (this allows you to restrict the families in some ways, for example for `HashSet`, only allow `T: Hash + Eq`). `A` will be the input type, and `B` will be the output type.

```rust
fn map<F>(self, this: This<Self, A>, f: F) -> This<Self, B>
where
    F: Fn(A) -> B + Copy;
```

Next we have this monstrous function! This will be easier to expain if we have a concrete family.

```rust
/// for `OptionFamily`
fn map<F>(self, this: Option<A>, f: F) -> Option<B>
where
    F: Fn(A) -> B + Copy;
```

So this is just the same as the ordinary map in `Option`, except it requries `Fn`, instead of `FnOnce`. We could generalize over `Fn` and `FnOnce` if Rust had associated traits, but alas we don't, so we'll have to make do with `Fn` (I'll leave it to you to figure out a nice way to abstract over `Fn`, `FnMut` and `FnOnce`). All we've done is generalize it to *any* family, not just `Option`.

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

In this way you can implement most, if not all `HKT` abstractions in stable Rust. However the ergonomics of these abstractions are downright abysmal. Bounds, bounds, bounds *everywhere*. Hopefully we can solve this on nightly, find out next time ... 