---
layout: post
title: Zig is for me - is it for you?
date: 2023-04-04
permalink: /posts/is-zig-right/
---

There has been [a](https://matklad.github.io/2023/03/26/zig-and-rust.html) [ton](https://www.openmymind.net/Zig-Quirks/) [of](https://www.infoworld.com/article/3689648/meet-the-zig-programming-language.html) [Zig](https://zackoverflow.dev/writing/unsafe-rust-vs-zig/) [content](https://kristoff.it/blog/zig-multi-sequence-for-loops/) lately. So I'll add to the swarm beating this dead horse, but only because I want to introduce some stuff that I'm working on :)

Also, I wanted to write more off-the-cuff style posts.

## Zig?

[Zig](https://ziglang.org/) is a modern programming language meant to "replace C." There are a ton of those. There does seem to be a worrying trend where every programmer under the sun believes that they will be the one to destroy C. Despite that, I see legacy C code every day that is run in safety-critical contexts. That code is *untouchable*. We all know that the second you touch something, it all will crumble. So why would you look at Zig?

There are a lot of reasons. One of the posts I linked above probably does a better job than I will at explaining this. Let's go through some requirements in a C successor:

1. A successor must have simple, easy control over memory. I believe this one of the main reasons C is so prominent - few other languages let you do what C does with memory with such simplicity. Most C applications work so closely with buffers and hardware that this is essential.
2. A successor must do number 1) more safely than C. If you've ever touched a C codebase, you've seen crazy segfaults where one corner of the application doesn't properly set memory where the other side assumes it's set. To make a language more appealing than C, you must make it easier to be safe.
3. A successor must be able to interact easily with C. If you are looking for a language to take over C, you probably have a lot of C sitting around. You don't want to rewrite all of that.
4. A successor must be able to compile to a lot of hardware, including proprietary hardware. Many devices you see use C for firmware. Many of these have no other choice. The ability to whip up a C compiler for your specific hardware is very important.
5. A successor must not be ugly. Ugly code isn't fun to write.

How does Zig deal with each of these?

1. Zig gives you control over [allocators](https://ziglang.org/documentation/master/std/#A;std:mem.Allocator) to chose how and where memory gets allocated. You also have access to suspiciously C like control over pointers...
2. Zig still requires manual memory management. If you allocate something, you free it. Or, if another function returns allocated memory, you will free it. Zig's allocators help you do this. First, the allocator pattern is generally implemented by you passing an allocator into a function, so you know it may allocate memory. Programmers should also make it clear that their function allocates memory. Second, you can have special allocators, like [`std.testing.allocator`](https://ziglang.org/documentation/master/std/#A;std:testing.allocator), which will tell you if you have memory leaks in a unit test. Neat!
3. Zig has my favorite thing: a transpiler from C (I [really like](https://etyp.dev/posts/art-of-transpilers/) those). Just run `zig translate-c` and you get Zig code where you had C code. You can use this to include C libraries or upgrade your code base to Zig!
4. Zig is based on LLVM, so you get a lot of hardware for free. I'd worry a bit about this point, since it seems a lot harder to make a good Zig compiler than a good C compiler. It's not super common to make a brand new C compiler for each piece of hardware (that's what a compiler backend is for), but there certainly are specialized C compilers. Stay tuned for this one.
5. Zig is mostly pretty. You have to get over stuff like `.{}` for anonymous structs (I find the anonymous structs syntax satisfying). But some things are great. Say your program returns some enum that can be `RED`, `BLUE`, or `GREEN`. You can return just by saying `return .BLUE;` - I think that's really cute. No need to even specify the enum name! (which is really convenient considering the enum could be anonymous...)

## Zig is special

I admit, not everyone has my sensibilities. Zig isn't the next Python, and many people conflate high level web development with programming in general. So you may hate it. That's okay. Any worthy art has those who don't like it.

What contexts would you use Zig? Obviously wherever C would be used. That could be operating systems, microcontrollers, or even game development. Zig also seems to be adding good support for WebAssembly targets - great for efficient web code. If you're like me, it may also be a good language for compilers, as it gives you a no nonsense, quick language for a compiler or interpreter. Certainly better than Python for that, in my opinion.

Zig is beautiful. It has cool ideas, almost none of which I covered. Just take a look at it, I think it has room to make a splash in certain spaces.

## What's next?

I've done a few things in Zig so far, but not as much as I want.

My current pet project takes Zig and puts it on the Arduino. I've gotten LEDs blinking, LCD panels with easy to use libraries, reading from sensors, pulse width modulation... all of that. Under the hood it uses [microzig](https://github.com/ZigEmbeddedGroup/microzig) to provide the heavy lifting, I'm just providing an abstraction on top of that abstraction.

Right now, Zig is in a pretty unstable place (it is pre-1.0 after all). Libraries you use won't always work with a new release. But, I think this time is crucial to build libraries and tools. Even if the project isn't a brand new library, just using Zig is extremely helpful for the community. If you have a new project, consider using Zig, to give exposure to the language. Maybe not for your fancy new website, though, unless you're [DHH](https://dhh.dk/) (for all of you Rails lovers).

I plan on making [arduzig](https://github.com/evantypanski/arduzig) just as easy to use as the Arduino IDE for whatever Arduino projects you have. I will probably (?) write examples and more posts to make it as understandable as possible. Probably don't try it yet. But once Zig 0.11 is out and I've brought it up to date? Please.

By the way, take a look at the posts I linked in the first sentence if you haven't. They're better than this one.
