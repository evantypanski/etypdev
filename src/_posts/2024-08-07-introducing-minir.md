---
layout: barebones
title: Introducing minIR
date: 2024-08-07
permalink: /posts/introducing-minir/
toc: true
---

{% include embed-audio.html src="/assets/audio/posts/introducing-minir/introducing-minir.mp3" title="Introducing minIR" %}

minIR is a compiler intermediate representation (IR) that's meant to be tiny and easy to work with.

You can find the [source code](https://github.com/evantypanski/minir) or an [example pass over it](https://github.com/evantypanski/minirpass) on [my Github](https://github.com/evantypanski).

## The Language

minIR has variables, branches, and labels, but no if statements and other higher level constructs. There's the example in the README that counts from 42 to 59:

> count_42_to_59.min
{:.filename}
``` cpp
func main() -> none {
  let i: int = 42
@loop
  test(i)
  i = addone(i)
  brc loop !(i == 60)
  ret
}

func addone(i: int) -> int {
  ret i + 1
}

func test(k: int) -> none {
  debug(k)
  ret
}
```

Then you can run it with the generated `minir` executable:

>
{:.shell}
```
etyp:minir/ $ zig-out/bin/minir interpret count_42_to_59.min
```

I could go through and introduce the language, but it may have pretty big changes. Instead, I'll abstain and just try to go over the more important parts of its design.

## Why minIR?

minIR is my direct response to a problem: I want to learn more about a lot of things in compilers like static analysis, optimizations, and just-in-time compiling. But, most IRs that I could work with come in one of two varieties:

1) A teaching IR (like [bril](https://github.com/sampsyo/bril)), which intentionally limits its scope in order to allow for students to implement certain details.

2) A giant IR (like [LLVM](https://github.com/llvm/llvm-project/tree/main/llvm)), which gives a bunch of awesome tools, but has a lot of overhead and requirements to get your feet wet.

I want an in between. In particular, I want a million different ways to inject whatever hacky script I wrote to optimize an IR. I also want that IR to be small enough that I can be reasonably confident it won't take a year to get a "production ready" analysis.

So, minIR is just an IR with a focus on developer experience, not on the actual language features. Many "hobbiest" languages explore one topic very thoroughly, but maybe they have bad error messages (or something else underdeveloped). That's completely understandable - most of these are made in someone's free time, the interesting part are the language features! But the developer experience *is* the interesting part of minIR.

### Developer experience + bad language = ?

A language is intertwined with its developer experience. If the language is not implemented fully (say, like current minIR doesn't have any structs or user-defined types), that necessarily hurts its developer experience. Then, we expect many things in modern languages: generics, user-defined types, casting... and those aren't implemented in minIR yet.

So part of me worries about that. The point, however, is not that I'm planning on making this big awesome ecosystem out of the box. In order to get to Rust levels of developer experience, with beautiful error messages and a million tools, you need years upon years. This is the case even for a small language!

I'm *not* saying minIR is the best developer experience out of any programming language (or IR). Instead, at any particular point in the language's life, it has a better-than-expected developer experience. I am prioritizing developer experience, that doesn't mean that it is the best developer experience over established art.

## Architecture

### Interpreting

minIR has two interpreters:

1) A tree-walking interpreter. This will walk down the tree after parsing (and whatever other passes are run) without another intermediate format.

2) A bytecode interpreter. This is a separate system which minIR gets lowered to then executed in.

This, hopefully, will help keep minIR more standard. Edge cases may get caught in one but not the other, so all implementations should be able to have a standard "output" for these edge cases more easily.

### Parsing

The parser is a handwritten [Pratt parser](https://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/). I got the idea for Pratt parsers (or "operator precedence" parsers) from [Crafting Interpreters](https://craftinginterpreters.com/) - really a must read for anyone working on a compiler. If you want a bit more about why, well just read the article or the whole book I've linked :)

### AST

The abstract syntax tree (AST) is all implemented using Zig's [tagged unions](https://ziglang.org/documentation/0.13.0/#Tagged-union). Let's look at a `Value`, which is (basically?) an expression:

> value.zig
{:.filename}
``` source.zig
const ValueTag = enum {
    undef,
    access,
    int,
    float,
    bool,
    unary,
    binary,
    call,
    type_,
    ptr,
};

// ...

pub const ValueKind = union(ValueTag) {
    undef,
    access: VarAccess,
    int: i32,
    float: f32,
    bool: u1,
    unary: UnaryOp,
    binary: BinaryOp,
    call: FuncCall,
    type_: Type,
    ptr: Pointer,

    // ...
};

pub const Value = struct {
    val_kind: ValueKind,

    // ...
};
```

Some code above has been emitted for brevity

This is the same concept as enums in Rust, but maybe with fewer pattern matching capabilities. You may also know them by other names like variants or sum types. They're often used when a language doesn't have inheritance. Instead of a `BinaryOp` inheriting from `Value`, we make `Value` possibly contain a `BinaryOp` in its `val_kind` field.

This pattern is pretty intuitive for me, but it comes with some drawbacks. The most annoying part within minIR has been making a visitor to walk over the tree, which...

### Visitors

Most operations will use a visitor in order to easily traverse the tree. This is provided in an `IrVisitor`:

> visitor.zig
{:.filename}
``` source.zig
pub fn IrVisitor(comptime ArgTy: type, comptime RetTy: type) type {
    return struct {
        // ...
    };
}
```

First, this syntax is a function which returns a type. It's how [Zig implements generics](https://ziglang.org/documentation/0.13.0/#struct). `IrVisitor` can use `ArgTy` in order to refer to the argument type used throughout the returned struct. Then `RetTy` is used to refer to the return type used throughout. You'll call the function with those two types, then use it like any other struct:

> blockify.zig
{:.filename}
``` source.zig
const VisitorTy = IrVisitor(*Self, Error!void);
```

This pattern, where a function takes in types as parameters and returns a type, is one of the more important concepts in Zig. It's a simple concept that produces powerful results. But, you lose the ability to refer to `IrVisitor` as a type without those two arguments, which can be frustrating.

When you have that visitor type, you can then initialize it and "overload" any functions you want to intercept:

> blockify.zig
{:.filename}
``` source.zig
pub const BlockifyVisitor = VisitorTy {
    .visitDecl = visitDecl,
};
```

This will use the internal `visitDecl` rather than the default `visitDecl` within the visitor, just make sure it is compatible with the function type provided by the visitor:

> visitor.zig
{:.filename}
``` source.zig
const VisitDeclFn = *const fn(self: Self, arg: ArgTy, decl: *Decl) RetTy;
```

Now that you can walk over the IR, how do you run a pass?

### Passes

A Pass is just a way to get your hands on the IR. You have three choices (at the time of writing), defined in `pass.zig`:

1) `Verifier`: These passes may only return an error or void. They *cannot* modify the AST

2) `Modifier`: These passes may only return an error or void. They *can* modify the AST

3) `Provider`: These passes may return an error or an arbitrary return value. They *cannot* modify the AST.

You also have another option, namely `SimplePass`, which is a wrapper around a `Verifier` but without the extra fluff that you may want for a verifier (like an initialization function).

When you use these passes, you need a `PassManager` (in `pass_manager.zig`). This object is created for you when using the standard `Driver` interface. If you're directly interacting with it, then all you'll need to do (once you have the manager) is that you must run `get`:

> pass_manager.zig
{:.filename}
``` source.zig
pub fn get(self: Self, comptime PassType: type) PassType.RetType {
```

Then you'll get your pass. So if you have a pass type (made from one of the four options above), you just call `pass_manager.get(MyPass)` and you'll get its return value. Easy.

To me, this is the most crucial part of minIR. It should be easy to create a pass and run that pass on the IR in whichever way you see fit. That's the goal here: flexibility plus simplicity.

### Testing

I've added a few easy cases for tests. The two notable ones are output tests, which make sure the output from running the program match the output provided (and the two interpreters match), and UI tests, which make sure things like error messages are output correctly.

Generally you'll use output for end-to-end testing to make sure the interpreters work together well, or UI tests to make sure error messages look a certain way.

In order to add a test, you just make a file in the proper directory within `tests`. Then, you go in `src/test/output.zig` or `src/test/ui.zig` and add the new test to the list of tests (this is using the [zig `test` declarations](https://ziglang.org/documentation/0.13.0/#Test-Declarations)). For UI tests, the necessary `.stderr` and `.stdout` will be created for you on the test's first run. For output tests, you'll only have a baseline if you explicitly provide a `.expected` file alongside the test. Otherwise, the test just ensures the two interpreters match.

There are a few different ways a test can fail.

First, there may be some memory error. Zig gives a testing allocator in `std.testing.allocator` that, when used in a test, will make it fail if a memory leak or other similar problems are detected.

Second, when you have a mismatch in program output, you also get a failure:

>
{:.shell}
```
error: 'test.output.test.output examples' failed: ====== expected this output: =========
3
7
4
2
42 oh no
␃

======== instead found this: =========
3
7
4
2
42
␃

======================================
First difference occurs on line 5:
expected:
42 oh no
  ^ ('\x20')
found:
42
  ^ ('\x0a')
```

I find that output pretty nice!

That's mostly it. There are a few caveats, namely that modifying the `.min` test itself won't trigger the cached test results to be invalidated, so you need to change a source file to get them to rerun. Also, the UI tests actually create a `minir` process call and use that, so you lose some benefits of the testing allocator for those. The reason for that is Zig-specific, I may just be unfamiliar with the language's testing tools.

## Where is minIR at?

Right now, you can kind of use minIR for simple things. If you want a simple dead code elimination pass, you can do that today, and it's pretty straightforward.

### Language

Beyond that you might start to see some issues. There are a lot of small things in the language that aren't quite up to the standard I want to be. As a small example, every function needs `ret` at the end of its body, even if it returns nothing - there are no implicit returns.

Another point is error messages. I have a somewhat decent interface using the `diag.err` method:

> typecheck.zig
{:.filename}
``` source.zig
self.diag.err(error.InvalidType, .{@tagName(ty), "!"}, val.*.loc);
```

Then we can find the format string used for `InvalidType` in the diagnostics engine:

> diagnostics_engine.zig
{:.filename}
``` source.zig
error.InvalidType => "{s} is an invalid type for '{s}'",
```

So the interface is decent, I just don't use it everywhere. I want to go through and add errors and warnings where necessary.

The language itself should be a little less minimal than what I've made. Some core points I think are casting and user defined types. Also, there should be more sizes of certain types (like 8, 16, 32, 64 bit integers). Some sort of string would probably be useful, too.

### Interface

The directory structure makes sense within minIR right now, but when using a `.zig.zon` file... it uses this crappy [lib.zig](https://github.com/evantypanski/minir/blob/main/src/lib.zig) file which just splattered what I needed in there, so that should be fixed up.

I also want easier ways to interact with minIR. One would be you should be able to use whatever language you want to write passes. I can use a JSON representation, but I really dislike JSON for serializing programming language ASTs. It may be best, though.

Then there are lots of simple things that can be better, like a better formatter.

### Bytecode

Right now the stack in the bytecode is just an array of `Value` - I want to make this variable sized, so each `Value` puts its tag first, then the size of the next bytes is determined from that tag. A value will then just be an access into a byte array, I think.

The bytecode should be printable. You should be able to run from just the bytecode and not interact with minIR.

Constants should be dealt with better - right now, all constants are just added to the bytecode, even if they already appear:

> chunk.zig
{:.filename}
``` source.zig
pub fn addValue(self: *Self, value: Value) !u8 {
    const idx = self.values.items.len;
    if (idx > std.math.maxInt(u8)) {
        return error.TooManyConstants;
    }
    try self.values.append(value);
    return @intCast(idx);
}
```

That should maybe [intern](https://en.wikipedia.org/wiki/Interning_(computer_science)) the values, instead.

### Misc

Part of minIR is that it should be small enough to fit in a [spec](https://github.com/evantypanski/minir/tree/main/spec) - that's for two reasons. First, it keeps it small and formal, so parsing a certain way can be either a bug or intended. Second, it forces acknowledgment of edge cases: what happens with an integer overflow? That sort of thing.

## Conclusions

Overall, I think minIR is far from done, but just about far enough along that I'm comfortable talking about it. I think it solves a relatively unique problem and has some interesting ideas about to solve those problems.

I don't really expect anyone to use it yet, but writing this is a good way to set my intentions straight and move forward making something interesting for others to use. Look for more minIR news in the coming while.
