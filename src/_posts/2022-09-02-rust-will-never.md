---
layout: post
title: Rust will `never` do that!
date: 2022-09-02
permalink: /posts/rust-will-never/
---

That is, the Rust [`never` type](https://doc.rust-lang.org/reference/types/never.html) (`!`).

## Everything is an expression

Rust is kind of an "everything is an expression" type of language. An expression is something that produces a value. Not everything is an expression, but basically everything that is in a Rust body (executable code) is an expression. Actually, one thing that isn't are locals, like:

``` rust
let x = 5;
```

That is its own statement! But you can have an expression with a `let` statement, just take:

``` rust

if let Some(num) = func_that_returns_option() {}
```

The `let` statement in there isn't a local, it's a [`let` expression](https://doc.rust-lang.org/stable/nightly-rustc/rustc_hir/hir/struct.Let.html).

Let's go over a few other things that are also expressions:

``` rust
{ 1 }
```

``` rust
loop { break 1; }
```

``` rust
if true { 1 } else { return; }
```

This last one is what we'll be interested in.

## The `never` type

So if we look into the last example I gave, you know the `if` block evaluates to `1`. But what about the `else` block? `return;` doesn't produce a value, it actually leaves the function. So what is its type?

That's right, the `never` type!

The point of the `never` type is to say "this computation will not complete." That way, if you assign something to that value:

``` rust
let x = if true { 1 } else { return; }
```

The type of `x` is based on the `if` block. But, that doesn't mean the `never` type is ignored. How the compiler figures that out is `never` can be coerced into any type. That means, when trying to infer the type for `x`, the compiler sees the `if` block has type `i32`. Then the `else` block has type `never`, so to make it consistent, it says "`never` can convert to `i32`" and it's all okay!

## What does this mean for the user?

Well, not much. It's mostly a fun compiler intricacy. But, if you want to see it mentioned in your diagnostics, try the new `let else` syntax (currently only available with nightly). This syntax lets you use a pattern for a `let` binding that may not always be true, like:

``` rust
let Ok(x) = returns_result() else { warn!("This is bad!"); return; };
```

So, if `returns_result` returns `Ok`, then we assign `x` to the value in `Ok`. But, if it's not, we warn and return. Pretty easy!

The type of that `else` block has to be `never`, though.

So try making it not, with this minimal example:

``` rust
#![feature(let_else)]

fn opt() -> Option<i32> {
    Some(1)
}

fn main() {
    let Some(x) = opt() else { 1 };
}
```

This will error:

``` rust
error[E0308]: `else` clause of `let...else` does not diverge
 --> letelse.rs:7:30
  |
7 |     let Some(x) = opt() else { 1 };
  |                              ^^^^^ expected `!`, found integer
  |
  = note: expected type `!`
             found type `{integer}`
  = help: try adding a diverging expression, such as `return` or `panic!(..)`
  = help: ...or use `match` instead of `let...else`

error: aborting due to previous error

For more information about this error, try `rustc --explain E0308`.
```

We expect that `else` block to diverge, so it has to be the `never` type!

Cool.

Rust will `never` return from that.
