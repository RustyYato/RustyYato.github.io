---
layout: post
title:  "Generalizing over Generics in Rust (Part 1.5): Mechanisms"
date:   2021-02-22 12:00:00 -0000
categories: type system,type families
---

Do you want to use Higher Kinded Types (`HKT`) in Rust? Are you tired of waiting for `GAT`? Well, this is the place to be.

This is part 1.5 in a series about emulating Higher Kinded Types in Rust, I'd encourage you to read [Part 1]({% post_url 2021-02-15-Type-Families-1 %}) for background before reading this.

I'd like to give credit where credit is due. I first heard of the idea of type families from nikomatsakis' blogs on [type families](https://smallcultfollowing.com/babysteps/blog/2016/11/03/associated-type-constructors-part-2-family-traits/) and [higher kinded types](https://smallcultfollowing.com/babysteps/blog/2016/11/04/associated-type-constructors-part-3-what-higher-kinded-types-might-look-like/) from back when GATs was called ATCs. In the comments of Part 1, /u/LPTK shared some literature on this topic from OCaml: [Lightweight Higher Kinded Polymorphism](https://www.cl.cam.ac.uk/~jdy22/papers/lightweight-higher-kinded-polymorphism.pdf), what we are doing is called type-level defunctionalisation[<sup>1</sup>](#note_1){: #note_1_back } ([comment link](https://www.reddit.com/r/rust/comments/ll9un4/generalizing_over_generics_in_rust_part_1_aka/gnxuz0y?utm_source=share&utm_medium=web2x&context=3)). /u/Krautoni also shared that [Arrow-KT](https://arrow-kt.io/) also uses defunctionalisation to implement `HKT` atop Kotlin's type system and GHC uses defunctionalisation to lower ~~redacted~~[<sup>2</sup>](#note_2){: #note_2_back } to GHC core. All of these are very interesting, and are definitely worth reading.

In [Part 1]({% post_url 2021-02-15-Type-Families-1 %}) I introduced the idea of type families, but without much explanation about *why* they worked, let's dig in now!

```rust
// `Option` has the kind `Type -> Type`,
// we'll represent it with `OptionFamily`
struct OptionFamily;

// This trait represents the `kind` `Type -> Type`
pub trait OneTypeParam<A> {
    // This represents the output of the function `Type -> Type`
    // for a specific argument `A`.
    type This;
}

impl<A> OneTypeParam<A> for OptionFamily {
    // `OptionFamily` represents `Type -> Type`,
    // so filling in the first argument means
    // `Option<A>`
    type This = Option<A>;
}
```

# Zero-Sized Tokens

The first part is

```rust
// `Option` has the kind `Type -> Type`,
// we'll represent it with `OptionFamily`
struct OptionFamily;
```

Where we use a zero-sized type as a token for some idea. This is actually a common feature of Rust, this is how functions are implemented. For example:

```rust
fn foo() {}
```

`foo` is a zero-sized value that has a unique type (In errors it's shown as `fn() {foo}`). Note this *isn't* `fn()`, a function pointer, but some anonymous zero-sized type. Closures are implemented in [a similar way]({% post_url 2019-01-17-Closures-Magic-Functions %}). This allows Rust to minimize the size of objects, and more easily optimize away function and closures calls (For example, in iterator chains). We can also use this idea of zero sized tokens to represent some global resource, for example a [`Global`](https://doc.rust-lang.org/alloc/alloc/struct.Global.html) allocator.

Using `rustc`'s notation, `OptionFamily` is being used to represent the idea of `fn(Type) -> Type {Option}`. A function that takes a `Type` and returns the a `Type` (Which happens to be `Option<InputType>`).

# Traits are Functions Declarations and Impls are Function Definitions

Here I think it's nice to first take a look at Haskell[<sup>3</sup>](#note_3){: #note_3_back }.

```haskell
-- Function declaration
one_param :: Family -> Type -> Type

-- Function definition
one_param OptionFamily a = Option a
```

```rust
// function declaration
pub trait OneTypeParam<A> {
    type This;
}

// function definition
impl<A> OneTypeParam<A> for OptionFamily {
    type This = Option<A>;
}
```

O.o how similar! This is the core idea behind generic meta-programming in Rust (I bet you didn't see that coming\![<sup>4</sup>](#note_4){: #note_4_back }). Here's the core idea, non-generic traits represent a function with a single argument: the type that they are implemented on. Each generic parameter increase the number of arguments to the function by one. Each item in the trait is an output.

So by having one generic parameter and one associated type, `OneTypeParam` has two functional parameters and one return value, just like `one_param` in Haskell! Also like in Haskell, this definition is split from the declaration.

However there is one minor difference and one major difference. The minor difference: in Rust you can have multiple "return" types. Just add more associated types. Each return type also has a name. In Haskell, this can be modelled using records, so the difference isn't significant, but it's worth mentioning. The major difference: the Rust type-level syntax is ***way*** more verbose. This makes anything but the simplest generic meta-programs almost inscrutable, so I don't recommend using this unless you need to.

Earlier in this post I said:

> and GHC uses defunctionalisation to lower ~~redacted~~ to GHC core

> The actual comment is a spoiler for what's upcoming in this post! I'll fill in the blanks afterwards :).

Well, let's fill in the blanks, spoiler's over!

> and GHC uses defunctionalisation to lower **type classes** to GHC core

Since `trait`s are Rust's version of type classes in Haskell, this makes sense. They are doing this same thing, so it makes sense that type classes are implemented like type-level functions in Haskell.

# Result: Type -> Type -> Type

Let's tie this up in a bow with `Result`. In Part 1 I said:

> * `Result` has the kind `Type -> Type -> Type`

So if we want to model `Result` in it's most general fashion, we need something like

```rust
struct ResultFamily;

trait TwoTypeParam<A, B> {
    type This;
}

impl<T, E> TwoTypeParam<T, E> for ResultFamily {
    type This = Result<T, E>;
}
```

What happens when we try to implement `ResultFamily` for `OneTypeParam`, just for shits and giggles?


```rust
impl<T, E> OneTypeParam<T> for ResultFamily {
    type This = Result<T, E>;
}
```

```rust
error[E0207]: the type parameter `E` is not constrained by the impl trait, self type, or predicates
 --> src/lib.rs:7:9
  |
7 | impl<T, E> OneTypeParam<T> for ResultFamily {
  |         ^ unconstrained type parameter

error: aborting due to previous error
```

![image tooltip here](https://images-wixmp-ed30a86b8c4ca887773594c2.wixmp.com/f/c25735ac-4a7c-4e2f-aee5-e80eeac4beb6/dcupirs-ebe369c0-db50-4879-a0a5-98b705c66522.png/v1/fill/w_1024,h_640,strp/surprised_pikachu_hd_wallpaper___remastered_by_thorofi_dcupirs-fullview.png?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1cm46YXBwOjdlMGQxODg5ODIyNjQzNzNhNWYwZDQxNWVhMGQyNmUwIiwiaXNzIjoidXJuOmFwcDo3ZTBkMTg4OTgyMjY0MzczYTVmMGQ0MTVlYTBkMjZlMCIsIm9iaiI6W1t7ImhlaWdodCI6Ijw9NjQwIiwicGF0aCI6IlwvZlwvYzI1NzM1YWMtNGE3Yy00ZTJmLWFlZTUtZTgwZWVhYzRiZWI2XC9kY3VwaXJzLWViZTM2OWMwLWRiNTAtNDg3OS1hMGE1LTk4YjcwNWM2NjUyMi5wbmciLCJ3aWR0aCI6Ijw9MTAyNCJ9XV0sImF1ZCI6WyJ1cm46c2VydmljZTppbWFnZS5vcGVyYXRpb25zIl19.BVqCJPQB-h2ld0gRfZElladNmrvk4rByFe2BRjUsjA8){:class="img-responsive" width="300" height="200"}

Ok rustc... I'll fix that error

```rust
// Note: in actual code you may want to use `fn() -> E`,
// to side-step drop check, but that's not too important for us
struct ResultFamily<E>(PhantomData<E>);

impl<T, E> OneTypeParam<T> for ResultFamily<E> {
    type This = Result<T, E>;
}
```

And that's how we get the implementation shown in Part 1. But why does this work? For the exact same reason as traits, adding a generic parameter to a struct increases it's functional parameters, so instead of `ResultFamily: Type` it is now `ResultFamily: Type -> Type`. If we look a the broader picture, we see that we've transformed `Result: Type -> Type -> Type` into `ResultFamily: Type -> Type`. This is currying in action! Just like doing the following in normal Rust.

```rust
fn curry_result(
    result: impl FnOnce(Type, Type) -> Type,
    e: Type
) -> impl FnOnce(Type) -> Type {
    let result_family = move |t| result(t, e);
    result_family
}
```

Of course we could curry the other parameter instead, like so:

```rust
// Note: in actual code you may want to use `fn() -> E`,
// to side-step drop check, but that's not too important for us
struct ResultFamily<E>(PhantomData<E>);

impl<T, E> OneTypeParam<E> for ResultFamily<T> {
    type This = Result<T, E>;
}
```

And depending on which one we use, we would have different semantics.

# Usage

I mentioned in the last post that using this method requires a lot of bounds. Let's see what I meant by that.

Let's write a function that composes two closures that return monads. Here we don't really care about *how* this is implemented (although I will provide the implementation in the end), but more about  the signature and everything required for that.

Let's start with the skeleton with a HKT-like syntax, and we'll reify it into our [definition]({% post_url 2021-02-15-Type-Families-1 %}#monad_definition) of `Monad` from part 1.

```rust
fn compose_monad<M, F, G, A, B, C>(
    monad: M,
    f: F,
    g: G
) -> impl FnOnce(A) -> M<C>
where
    F: FnOnce(A) -> M<B>,
    G: FnOnce(B) -> M<C>,
{
    move |a| f(a).bind(g)
}
```

First: of course this doesn't compile. It's not even valid Rust syntax. But this is what HKT could look like in Rust. Second: Lots of type parameters! Let's see what they all mean:

* M - What monad? `Option`, `Result<_, E>`, `Vec`, something else?
* F - The first closure that will be applied
* G - The second closure that will be applied
* A - The parameter for the first closure
* B - The parameter for the second closure, *and* what's contained in the monadic output of the first closure
* C - what's contained in the monadic output of the second closure

Now how to we reify this? First we know that `M` must be a monad, so let's introduce that bound, and `Monad` implies `OneTypeParam`, so we can use the [`This` type alias]({% post_url 2021-02-15-Type-Families-1 %}#this_alias_definition) to represent `M<A>`, `M<B>`, and `M<C>`.

```rust
fn compose_monad<M, F, G, A, B, C>(
    monad: M,
    f: F,
    g: G
) -> impl FnOnce(A) -> This<M, C>
where
    M: Monad<A, B> + Monad<B, C>,
    F: FnOnce(A) -> This<M, B>,
    G: FnOnce(B) -> This<M, C>,
{
    move |a| f(a).bind(monad, g)
}
```

Notice how we needed to declare `Monad` twice. This is unsatisfactory. It would be nice if we could just say the following and be on our way.

```rust
fn compose_monad<M, F, G, A, B, C>(
    monad: M,
    f: F,
    g: G
) -> impl FnOnce(A) -> This<M, C>
where
    M: Monad,
    F: FnOnce(A) -> This<M, B>,
    G: FnOnce(B) -> This<M, C>,
{
    move |a| f(a).bind(g)
}
```

Granted this is a small case, but even for such a minor function there is a lot of annotation. This annotation burden only gets worse as complexity increases.

# Next Time

Next time in Part 2, we'll properly introduce GATs and discover a way to reduce the number of required bounds when using `HKT` in Rust.

<sup>1</sup> Eh, functions at the type level, what are these people on about. Rust doesn't have functions at the type level /s. (We'll explore this in this post) ([back](#note_1_back))
{: #note_1 }

<sup>2</sup> The actual comment is a spoiler for what's upcoming in this post! I'll fill in the blanks afterwards :). If you are fine with a minor spoiler, here's the [comment link](https://www.reddit.com/r/rust/comments/ll9un4/generalizing_over_generics_in_rust_part_1_aka/go5n2ux?utm_source=share&utm_medium=web2x&context=3) ([back](#note_2_back))
{: #note_2 }

<sup>3</sup> Although I'm using Haskell here, I recommend you use Proglog to model out your generic meta-programming. It's far closer to how the trait solver actually works, so it will give better feedback on what's possible and what isn't. ([back](#note_3_back))
{: #note_3 }

<sup>4</sup> Unless of course you are /u/Michael-F-Bryan! who mentioned how close this felt to template meta-programming in C++ ([back](#note_4_back))
{: #note_4 }