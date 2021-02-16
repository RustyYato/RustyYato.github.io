---
layout: post
title:  "PUI Part 1: Core"
date:   2020-01-20 12:00:00 -0700
categories: pui
---

Welcome to this series on [`pui`](https://crates.io/crates/pui), a library that provides **p**rocess **u**nique **i**dentifiers (PUIs). What are PUIs? Read on to find out. If you want to see the entire series, check out the [introduction](../18/PUI-Introduction.html) for links to each part.

This part will focus on [`pui-core`](https://crates.io/crates/pui-core) major traits and [`pui-core::scoped`](https://docs.rs/pui-core/0.5.2/pui_core/scoped/index.html). 

## Major Traits

Let's start with the three major traits of [`pui-core`](https://crates.io/crates/pui-core): [`Identifier`](https://docs.rs/pui-core/0.5.2/pui_core/trait.Identifier.html), [`Token`](https://docs.rs/pui-core/0.5.2/pui_core/trait.Token.html), and [`OneShotIdentifier`](https://docs.rs/pui-core/0.5.2/pui_core/trait.OneShotIdentifier.html).


### [trait Token](https://docs.rs/pui-core/0.5.2/pui_core/trait.Token.html)

```rust
pub unsafe trait Token: Clone + Eq { }
```

Token is an unsafe marker trait that has two requirements to be safely implemented.

1. `Eq` is reflexive
2. all clones are equal to each other
3. `PartialEq::eq`'s behavior can't change if you only have access to a shared reference to a token

Why are these requirements the way they are? We'll need to peek into `Identifier` to see...


### [trait Identifier](https://docs.rs/pui-core/0.5.2/pui_core/trait.Identifier.html)

The most important trait [`pui`](https://crates.io/crates/pui) is `Identifier`. It's main purpose is to specify what it means to be a PUI.

It has a few interesting parts, a `Token`, and two functions for operating on tokens.

```rust
pub unsafe trait Identifier {
    type Token: Token;

    pub fn token(&self) -> Self::Token;

    pub fn owns_token(&self, token: &Self::Token) -> bool { ... }
}
```

Note: It's unsafe to implement `Identifier`, because we want to rely on it's guartnees in `unsafe` code. If `Identifier` was not unsafe to implement, we would not be able to do that.
In particular [`pui-cell`](https://docs.rs/pui-cell/) and [`pui-vec`](https://docs.rs/pui-vec/) require that implementations of `Identifier` be correct in order to provide memory safety.

What does it mean to provide a correct implementation of `Identifier`

* the tokens produced by `Identifier::token` can only be equal to other tokens produced by other calls to the *exact same* `Identifier`
* `Identifier::owns_token` can't return true for any token that was produced by a different currently live `Identifier`

This is a rather complex requirement. It's also why `Token` must be `unsafe`. The only way we can compare identifiers to see if they are the same identifier, is to compare their tokens. IT may be easier to understand the second requirement is by looking at a piece of code that boils it all down.

This function captures all of the requirements for a safe implementation of `Identifier`: For all types `A` and `B` that can be passed it this function, this function should ***never*** trigger any of the three asserts (no matter what order the asserts appear in).

```rust
fn check_identifier_validity<A, B>(a: A, b: B)
where
    A: Identifier,
    B: Identifier<Token = A::Token>,
{
    let a_token = a.token();
    let b_token = b.token();

    assert_ne!(a_token, b_token);
    
    assert!(!a.owns_token(&b_token));
    assert!(!b.owns_token(&a_token));
}
```

These two identifiers are different, because we have two instances. So their tokens must also be different. This check should ***always hold***, even if the tokens are cloned/copied.

Here we can finally see why `Token` has it's requirements.

1. `Eq` is reflexive
    * This one is more obvious, we want tokens to be equal to themselves, otherwise it would be too hard to do analysis
2. all clones are equal to each other
    * This allows you to get clones of tokens, and still figure out that they are all owned by the same identifier, again without this it would be too hard to figure out where a specific token came from.
3. `PartialEq::eq`'s behavior can't change if you only have access to a shared reference to a token
    * This one is required so that `Clone` doesn't change the behavior of `PartialEq::eq` in any way. This is largely only possible if you use `UnsafeCell` or types built atop it.

These requirements on `Token`, you to be sure that tokens can be used to identify `Identifier`s. Note that any type that implements `StructuralEq` and doesn't do anything fancy for `Clone` can safely implement `Token`. So these requirements aren't difficult to meet.