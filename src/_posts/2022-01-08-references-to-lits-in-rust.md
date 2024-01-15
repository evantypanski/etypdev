---
layout: barebones
title: References to Literals in Rust?!
date: 2022-01-08
permalink: /posts/references-to-literals-in-rust/
---

One day messing around with Rust, I found that the following code is valid:

``` rust
fn main() {
    let x = &0;
}
```

That's assigning a variable to a reference to the literal `0` - how?! Why?! This absolutely shocked me. Just try doing this in C++ and you'll see why:

``` rust
error: non-const lvalue reference to type 'int' cannot bind to a temporary of type 'int'
    int &r = 0;
         ^   ~
```

The literal is a temporary - you can't have a reference to that! String literals are lvalues in C++, but that's a weird special case. That's why you can assign it to a pointer like `const char *`, but can't get a `const int *` from an integer literal.

## Why is this shocking?

This may not seem that shocking to some. Literals are generally temporary and don't really live anywhere in memory - they're essentially hard coded constants in the program. A reference points to some place in memory. How do we point to something that doesn't live in memory? Well, we can't, and we don't!

## Rvalue Static Promotion

This concept in Rust is called [rvalue static promotion](https://rust-lang.github.io/rfcs/1414-rvalue_static_promotion.html). We can look at each part to see what that means:

Rvalue: Something that can only be on the right hand side of an assignment. For example, you can't do `1 = x` because the literal `1` is an rvalue.

Static: Something that is valid for the whole lifetime of the program.

So we promote the rvalue to a static value in order to take a reference to it. Looking at the program earlier, we can see this in action in [Rust's playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=9fe3d6deffc976b8394163f6008b9093). We can see the MIR (one of the intermediate representations of Rust) is:


``` rust
fn main() -> () {
    let mut _0: ();                      // return place in scope 0 at src/main.rs:1:11: 1:11
    let _1: &i32;                        // in scope 0 at src/main.rs:2:9: 2:10
    let mut _2: &i32;                    // in scope 0 at src/main.rs:2:13: 2:15
    scope 1 {
        debug x => _1;                   // in scope 1 at src/main.rs:2:9: 2:10
    }

    bb0: {
        _2 = const main::promoted[0];    // scope 0 at src/main.rs:2:13: 2:15
                                         // ...
        _1 = _2;                         // scope 0 at src/main.rs:2:13: 2:15
        return;                          // scope 0 at src/main.rs:3:2: 3:2
    }
}

promoted[0] in main: &i32 = {
    let mut _0: &i32;                    // return place in scope 0 at src/main.rs:2:13: 2:15
    let mut _1: i32;                     // in scope 0 at src/main.rs:2:14: 2:15

    bb0: {
        _1 = const 0_i32;                // scope 0 at src/main.rs:2:14: 2:15
        _0 = &_1;                        // scope 0 at src/main.rs:2:13: 2:15
        return;                          // scope 0 at src/main.rs:2:13: 2:15
    }
}
```

This is a little weird to look at if you've never seen MIR before, but the important part is the line `promoted[0] in main: &i32` - that's where we see the promoted variable! Then in the main program we assign with `_2 = const main::promoted[0];`. So we lift the literal out to a static lifetime in order to return a reference, pretty neat.

### Why did they do this?

I find this the interesting part. We can see a lot of the motivation for this in the feature:

> The necessary changes in the compiler did already get implemented as part of codegen optimizations (emitting references-to or memcopies-from values in static memory instead of embedding them in the code).

It seems like it was just an easy thing to implement, so they did it. Their drawback is pretty interesting:

> One more feature with seemingly ad-hoc rules to complicate the language...

I found this funny. Seems like they just thought "it's easy enough, could be useful, why not?" So, they added a new feature to the Rust language. So many languages get by without this, but the Rust devs said, why not?

### It's useful!

You can see this exact thing in action in Rust's source code! At the time of writing, you can see this [here](https://github.com/rust-lang/rust/blob/8b09ba6a5d5c644fe0f1c27c7f9c80b334241707/compiler/rustc_borrowck/src/nll.rs#L74):

``` rust
    dump_mir(infcx.tcx, None, "renumber", &0, body, |_, _| Ok(()));
```

The fourth parameter is a reference to the literal `0`!

Well, I don't know how useful you'd say it is. But, it's an interesting thing in a common compiler that not many languages have.
