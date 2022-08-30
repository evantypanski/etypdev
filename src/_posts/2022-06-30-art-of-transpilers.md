---
layout: post
title: The Art of Transpilers - From One Programming Language to Another
date: 2022-06-30
permalink: /posts/art-of-transpilers/
---

Compilers are cool, they take code you write and make code the machine can understand. Transpilers are cooler, they take code you write and make code in another programming language that machines can't understand (yet). But why?

Let's take a quick tour around the world of transpilers!

## JavaScript Clones
JavaScript is one of the top use cases for transpilers. JavaScript can run natively in any major browser. So, instead of making a new language and asking browsers to support that natively in the browser, many want to reuse what's already there.

The problem is, [JavaScript is crazy](https://blog.andrasbacsai.com/javascript-is-going-crazy-1). There are many more examples of that. Even with perfect knowledge of JavaScript internals, bugs will happen because of just how dynamic the language is. So, to get around that, we have...

### TypeScript
[TypeScript](https://www.typescriptlang.org/) is a superset of JavaScript that allows explicit type information. TypeScript solves many of JavaScript's biggest woes with dynamic typing. I will not provide examples so I don't go insane.

So, how does TypeScript do it? It's a transpiler! You can just transpile your TypeScript to JavaScript then run *that* in the web browser.

So take this example in TypeScript, ruthlessly stolen from the [handbook](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html) on everyday types:

``` ts
// test.ts
// Parameter type annotation
function greet(name: string) {
  console.log("Hello, " + name.toUpperCase() + "!!");
}
```

So we can see our `name` parameter has the type `string`. We can then compile this with `tsc test.ts` to get...

``` js
// test.js
// Parameter type annotation
function greet(name) {
  console.log("Hello, " + name.toUpperCase() + "!!");
}
```

... The same thing without the type in `name`! Okay that's not interesting, but try adding a call like `greet(1);` and you'll see an error:

``` bash
test.ts:6:7 - error TS2345: Argument of type '1' is not assignable to parameter of type 'string'.

6 greet(1);
        ~


Found 1 error.
```

The type guarantee lets you call `toUpperCase()` and declines anything that's not a `string` type. You may be able to see the bugs that could be caught from this. That's why TypeScript has made such a splash.

### CoffeeScript

[CoffeeScript](https://coffeescript.org/) tries to solve a slightly different problem. Instead of providing a superset that makes your code less prone to bugs, CoffeeScript makes JavaScript prettier.

However, I've always considered CoffeeScript its own language. Once learned, it's a pleasure to write, which is a nice break from JavaScript, in my opinion. But, it seems to be in some middle ground between JavaScript and a standalone language.

In fact, as stated on the CoffeeScript site, NodeJS can natively run CoffeeScript. It doesn't need to transpile down to JavaScript; it is its own language. What if we went more pure, with no intent to replace the syntax of some other language?

## Nim

[Nim](https://nim-lang.org/) can transpile to C, C++, or JavaScript. That would make Nim our lone example of a language whose purpose is to transpile to multiple other languages. For systems programming, use its C support with dependency-free executables. For frontend development, target JavaScript.

Let's try an example:

``` num
// test.nim
echo "Hello World!"
```

If we compile this to C with `nim cc --nimcache:nimcache test.nim`, we get.. an executable. So did it compile? Or transpile?

Well, if we check the nimcache directory I specifically included, we'll find a fair few C files. These C files are what get compiled to create the executable we see. I won't include these, since they're a fair bit of code.
But, we see a pretty major difference from the JavaScript-like transpilers we looked at: Nim only uses the transpilation as a means to an end.

## C2Rust

But what if you want to permanently change your codebase? That's the idea behind [C2Rust](https://c2rust.com/). There's value in updating legacy codebases to safer, more secure languages like Rust. So, if you want to do it, try a transpiler!

The value in transpilers stretches far beyond just writing simpler code, or code that targets some other language for backend work. Transpilers can change your codebase and make modernizing the code far easier. In the process, your code becomes safer.

## Many frontends, one backend

Imagine you have a tool that you want to support many languages, like for static analysis. But, you don't want to write different analyses for every language's internal representation. What can you do? Introduce a new language, then transpile to that!

That's what I currently work on every day. I make translators that go from many source languages and produce one proprietary language as its output. Then, we run checks based on that internal language.

This can have a pretty high barrier to entry because you have to design a new language *and* your transpiler. But, depending on the use case, it may be an interesting solution.

Those familiar with LLVM may wonder why this is different from compilation? Many compilers, from Rust to Clang, generate LLVM from your source code. Then, the LLVM generates the machine code. But, that's a compiler, not a transpiler. Don't be silly.

# Why?
I don't really have profound conclusions about transpilers. I just think they're cool and underappreciated. And, when you work with them, you get to see the internals of two languages at once, rather than just one!

Two is better than one!

So write more transpilers... And compilers.
