---
layout: post
title: Writing a Simple Programming Language from Scratch - Part 3
date: 2021-12-30
permalink: /posts/writing-a-simple-programming-language-p3/
---

[Part 1](https://etyp.dev/posts/writing-a-simple-programming-language-p1/)

[Part 2](https://etyp.dev/posts/writing-a-simple-programming-language-p2/)

Source code found [here](https://github.com/evantypanski/ectlang), for much easier running and compiling.

We've gotten a compiler to bring code in our source language, ectlang, to a language most are familiar with, C++. Now, we're going to change our target to something a real compiler may use: LLVM!

## What's LLVM?

[LLVM](http://www.aosabook.org/en/llvm.html) is an [intermediate representation](https://en.wikipedia.org/wiki/Intermediate_representation) (or IR) of the source code. Think of the C++ code we wrote earlier: some other tool grabbed it from that point to get it down to the machine code we can run. Now, we'll target LLVM, a language that's meant to be used as an IR.

This is a common pattern in compilers because it allows for you to separate your concerns. The compiler we're writing is actually only the front-end: it changes based on the language (language specific), but doesn't change when we move it over to some other architecture (platform agnostic). LLVM is one of the most common ways to bring your compiler from its frontend to people's computers. Thus, we don't have to worry about targeting all of those different architectures, it's done for us!

And, luckily for us, LLVM has its own API, easily usable from C++.

## Using LLVM

There are many tutorials, such as the amazing [Kaleidoscope](https://llvm.org/docs/tutorial/), that does a much better job of explaining LLVM and what it can do for you. The goal for me is to stick our foot in the door. LLVM is actually pretty accessible - far more than assembly! It's just, at this point, we're writing code with code.

So, to start, you'll need to install LLVM to your computer - like on Ubuntu, you can use `sudo apt install llvm-12-dev`. If that doesn't work, just google it, it shouldn't be too hard. We'll take a break from our compiler for a bit and just compile some straight up C++ code.

I'm going to take this bit by bit, then dump the full code for our prototype at the end, just to not overwhelm anybody.

First, LLVM programs will start with three main objects: `LLVMContext`, `Module`, and `IRBuilder`. Here's about how you'll instantiate each:

``` cpp
// Context - Owns the types and constants we'll use.
llvm::LLVMContext context;

// Module - Contains functions and global variables that we generate.
std::unique_ptr<llvm::Module> mod;
mod = std::make_unique<llvm::Module>("ECTLang module", context);

// IR Builder - what we'll add instructions to. More of a helper class.
llvm::IRBuilder<> builder(context);
```

Understanding each of these isn't incredibly important, but I'll explain some. The context is where we'll grab, say, an int type, or float type, or the constant `1`. We treat it as a black box: it knows how to make things we want, that's about it.

The module stores the information that we make. So, if we make a function, that function is owned by the module.

The IR builder is just an object we use to make creating instructions for basic blocks easier. To create a basic block...

``` cpp
// Function returns void.
llvm::FunctionType *functionReturnType =
    llvm::FunctionType::get(llvm::Type::getVoidTy(context), false);

// Our main function.
llvm::Function *mainFunction =
    llvm::Function::Create(functionReturnType,
                           llvm::Function::ExternalLinkage,
                           "main",
                           mod.get());

// Now make a basic block inside of mainFunction.
llvm::BasicBlock *body = llvm::BasicBlock::Create(context, "body", mainFunction);
builder.SetInsertPoint(body);
```

Easy enough. We make a function, call it `main`, then make a new basic block with that function as its parent. Then, we let our builder know we're going to insert instructions in the new basic block.

Basic blocks are hugely important it control flow analysis and compilers. Rust explains them extremely concisely [here](https://rustc-dev-guide.rust-lang.org/appendix/background.html#cfg). All they are is a bunch of commands that run sequentially. When leaving a basic block, then it can jump to some other basic block based on whatever conditions. But, we're writing a language that doesn't even have `if` statements, we'll be fine without much more than knowing to put our instructions in a basic block.

Now, we have a function that contains a basic block. Let's write our first instructions!!

``` cpp
llvm::Constant *x = llvm::ConstantInt::getSigned(llvm::Type::getInt32Ty(context), 5);
llvm::Constant *y = llvm::ConstantInt::getSigned(llvm::Type::getInt32Ty(context), -6);
llvm::Value *add = builder.CreateAdd(x, y, "addtest");
```

And that's how you'd make a simple add instruction. Easy, right? The first argument get `getSigned` shows how the context is used to get types. To finally see your output LLVM, you can add a return statement and output the program to standard out:

``` cpp
builder.CreateRetVoid();
mod->print(llvm::outs(), nullptr);
```

If you run what we have so far (which is just a function with `5 + -6` in it) you get the following LLVM:

``` llvm
; ModuleID = 'ECTLang module'
source_filename = "ECTLang module"

define void @main() {
body:
  ret void
}
```

Our main function doesn't have anything resembling what we wrote in it! That's because we never actually added our add to the body. We're going to settle this by printing it.

First, we'll make the function prototype. Here's what `printf` looks like in C:

``` cpp
int printf(const char *format, ...)
```

And here's how we can put that in our LLVM program:

``` cpp
    std::vector<llvm::Type *> params;
    params.push_back(llvm::Type::getInt8PtrTy(context));
    llvm::FunctionType *printfType =
            llvm::FunctionType::get(builder.getInt32Ty(), params, /*isVarArg=*/true);
    llvm::Function::Create(printfType, llvm::Function::ExternalLinkage, "printf",
                       mod.get());
```

It's pretty trivial to use a search engine to find whatever function you may need's prototype. Just figure out how they map to LLVM and you have an easy job. For the function `llvm::FunctionType::get`, the first argument is the return type (`int`), the second is a vector of parameters (this can be omitted if there are no parameters), and the last is whether the function is variadic or not.

We created the actual prototype in our module with the function call `llvm::Function::Create`. The only really interesting part here is that we specified the function is externally linked (via `llvm::Function::ExternalLinkage`). That just specifies that the linker will find the correct function to use, given we gave the correct information to find it (that is, the prototype).

Now for the fun part, we need to output our add call to `printf`. You can construct this just like you would in C, just via code. We want to make the following:

``` cpp
printf("%d\n", 5 + -6);
```

And here's how we construct that function call in LLVM:

``` cpp
    std::vector<llvm::Value *> printArgs;
    llvm::Value *formatStr = builder.CreateGlobalStringPtr("%d\n");
    printArgs.push_back(formatStr);
    printArgs.push_back(add);
    builder.CreateCall(mod->getFunction("printf"), printArgs);
```

Pretty straightforward! We create an argument for the format string, an argument for the add instruction that we created earlier, then create the function call. The module holds the function that we need, so we can just grab it by name.

Finally, just add a return statement and output the program the standard output. Here's the full program, for reference:

``` cpp
// main.cpp
#include "llvm/IR/Module.h"
#include "llvm/IR/IRBuilder.h"

int main() {
    llvm::LLVMContext context;

    std::unique_ptr<llvm::Module> mod;
    mod = std::make_unique<llvm::Module>("ECTLang module", context);

    // Build IR for this block.
    llvm::IRBuilder<> builder(context);

    // Function returns void.
    llvm::FunctionType *functionReturnType =
        llvm::FunctionType::get(llvm::Type::getVoidTy(context), false);

    // Our main function.
    llvm::Function *mainFunction =
        llvm::Function::Create(functionReturnType,
                               llvm::Function::ExternalLinkage,
                               "main",
                               mod.get());

    // Now make a basic block inside of mainFunction.
    llvm::BasicBlock *body = llvm::BasicBlock::Create(context, "body", mainFunction);
    builder.SetInsertPoint(body);
    llvm::Type *intTy = llvm::Type::getInt32Ty(context);
    // Make a simple add with two constants.
    llvm::Constant *x = llvm::ConstantInt::getSigned(llvm::Type::getInt32Ty(context), 5);
    llvm::Constant *y = llvm::ConstantInt::getSigned(llvm::Type::getInt32Ty(context), -6);
    llvm::Value *add = builder.CreateAdd(x, y, "addtest");

    std::vector<llvm::Type *> params;
    // Pointer to int8 would be like char *
    params.push_back(llvm::Type::getInt8PtrTy(context));

    llvm::FunctionType *printfType =
            llvm::FunctionType::get(builder.getInt32Ty(), params, /*isVarArg=*/true);
    llvm::Function::Create(printfType, llvm::Function::ExternalLinkage, "printf",
                       mod.get());

    // Function call arguments
    std::vector<llvm::Value *> printArgs;
    llvm::Value *formatStr = builder.CreateGlobalStringPtr("%d\n");
    printArgs.push_back(formatStr);
    printArgs.push_back(add);
    builder.CreateCall(mod->getFunction("printf"), printArgs);

    builder.CreateRetVoid();
    mod->print(llvm::outs(), nullptr);

    return 0;
}
```

Now to compile this program, use your favorite compiler with some extra LLVM tooling help. When you install LLVM, you also get a tool `llvm-config` which will help with compiler and linker options for your LLVM install. You can use it with `GCC` like so (mine is called `llvm-config-12` for the LLVM version I installed):

``` bash
$ g++ main.cpp $(llvm-config-12 --ldflags --libs --cxxflags) -o testllvm.out
$ ./testllvm.out
```

Running the output, we get the following LLVM:

``` llvm
; ModuleID = 'ECTLang module'
source_filename = "ECTLang module"

@0 = private unnamed_addr constant [4 x i8] c"%d\0A\00", align 1

define void @main() {
body:
  %0 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([4 x i8], [4 x i8]* @0, i32 0, i32 0), i32 -1)
  ret void
}

declare i32 @printf(i8*, ...)
```

Being able to read that LLVM isn't important, but we can see the structure seems about right. We have a call to `printf` with `-1` as our add output being passed in. We get the result of the add as an argument because we used constants, which works for our super simple language well enough.

We can compile this LLVM directly with Clang into something we can run! Put that code in a file (I'll call it `test.ll`) and compile it:

``` bash
$ clang++ test.ll -o test.out
$ ./test.out
```

We see `-1` is printed, which happens to be the result of `5 + -6`!

Since our language has floats, let's also make sure we can do that. If we want to add two floats, it's actually pretty easy. We need to change `x` and `y` to be `double` types:

``` cpp
    llvm::Constant *x = llvm::ConstantFP::get(llvm::Type::getDoubleTy(context), 5.5);
    llvm::Constant *y = llvm::ConstantFP::get(llvm::Type::getDoubleTy(context), -6.5);
```

Switch our `add` call to be the floating variant:

``` cpp
    llvm::Value *add = builder.CreateFAdd(x, y, "addtest");
```

Then change our format string to accept `%f` to format it at as a float:

``` cpp
    llvm::Value *formatStr = builder.CreateGlobalStringPtr("%f\n");
```

And if we run/compile that like before, we get a similar result, but floaty:

```
-1.000000
```

## Generating LLVM

Now that we understand LLVM a bit better, we need to generate LLVM from our small compiler. We'll use the visitor that we made in part 2 to generate some LLVM!

I'm going to skip setup such as initializing the visitor with the required LLVM context and module, along with other minor details. I'll use code to explain concepts, but if you want working code, check out the Github repo at the top.

### Building "return" Values

Our visitor we made in the last part doesn't have a return type! Because of this, the more obvious pattern of returning the `llvm::Value` at each step isn't feasible. But, we can make each visit make a "promise" to fill a variable. Then, we can just treat that variable as the return value. Here's an example for the integer literal type:

``` cpp
// llvmvisitor.cpp
void LLVMVisitor::visit(IntegerNode *node) {
    // Return the LLVM int value.
    ret = llvm::ConstantInt::getSigned(llvm::Type::getInt32Ty(context), node->getValue());
}
```

Then, if we want to use that value in the visit function for plus:

``` cpp
// llvmvisitor.cpp
void LLVMVisitor::visit(PlusNode *node) {
    // Get the return value from the left side.
    node->getLeft()->accept(*this);
    llvm::Value *lhs = ret;

    // Get the return value from the right side.
    node->getRight()->accept(*this);
    llvm::Value *rhs = ret;

    // Return the add.
    ret = builder.CreateAdd(lhs, rhs);
}
```

And with that we can have add working! Just do that for the rest of the operations, then we come into another problem.

### Dealing with floats

Floats are weird. We can't add an int and a float natively in LLVM. So, instead, we're going to promote everything as soon as we see a float. The logic around this can be a bit more complicated, but we'll try to keep it simple for demonstration purposes. So, if we see a float, we need to promote both sides to a float, then use the float method associated with the operation (eg `builder.CreateFAdd`). This is what that may look like for add:

``` cpp
// llvmvisitor.cpp
    if (floatInst) {
        // Promote RHS or LHS if we're dealing with floats and they're not a float.
        // (except we use doubles)
        if (!lhs->getType()->isDoubleTy())
            lhs = builder.CreateSIToFP(lhs, llvm::Type::getDoubleTy(context));
        if (!rhs->getType()->isDoubleTy())
            rhs = builder.CreateSIToFP(rhs, llvm::Type::getDoubleTy(context));

        ret = builder.CreateFAdd(lhs, rhs);
    } else {
        // Otherwise we're just doing an integer add.
        ret = builder.CreateAdd(lhs, rhs);
    }
```

Easy! We have some variable `floatInst` which gets set when we see a float, then gets reset when we see a `StatementNode`. This tells us we need to promote the remaining values that are not floats and use float instructions. We check each side to see if it's a double (since we're setting the type to `llvm::Type::getDoubleTy(context)`, works the same with `Float` and `isFloatTy()`), then we promote if it's not.

Note, remember to change the format string in the `printf` call when using a float.

### A complete visitor

With that, we have a complete visitor! I put maybe a quarter of the work in this post, with the remaining work already explained, just slightly different. If it's unclear, feel free to check out the repo at the top.

Here's how you can compile it and run some program you make `source.ect`:

``` bash
$ bison parser.ypp
$ flex -o scanner.c scanner.lex
$ g++ main.cpp parser.tab.cpp nodes/*.cpp $(llvm-config-12 --ldflags --libs) $(llvm-config-12 --cxxflags) -o ectfrontend.out
$ ./ectfrontend.out < source.ect
$ clang++ test.ll
$ ./a.out
```

## This Language Sucks

Okay, now that we have a "working" language, I want to explain many of the reasons this language sucks. The fundamentals I tried to show in this post remain valid, but everything else is just a tool for getting those fundamentals out with the least amount of background knowledge I could muster. But, knowing some of the shortcomings could help us all not make these mistakes when we actually decide to build a compiler.

 - No semantic analysis
   - We don't check to see that the code is valid, other than it parses. Turns out this could not be horrible for our toy language, but checking for overflows, valid casts, etc. are super important for a compiler.
 - No warnings, diagnostics, assertions...
   - If your program is ill formed, you probably should tell the user why. Depending on how it's ill formed, our compiler may say "syntax error" or just segfault. That's pretty bad design.
   - We also don't have assertions. These are pretty useful in any program, so when you build debug you can make sure your assumptions are valid. For example, if we trust our assumption that `ret` is set for each arithmetic operation, we should add an assertion to express that assumption.
 - No unary minus
   - Yeah, you can't add `5 + -6`. Even though I use that as an example up when explaining the LLVM API. Oops. You can still have negative numbers, though?
 - We expose the IR
   - Not a big deal, but most people don't want to compile LLVM directly from a file afterwards. Ideally, you'd continue to use the API for LLVM (or whatever intermediate representation you're using) to create the executable.
 - Bad CLI
   - Our command line interface sucks. You have to pass the program directly in rather than just calling with the file name. You can't specify the output file's name. There's no command line options, even like `--help`. If anyone ever uses your CLI, these are not just helpful, they're necessary.
 - We abuse C++
   - I purposefully did this to rant about C++ a little. If you noticed, we actually assume that `ret` is set when we set the `lhs` and `rhs` variables. If it's not set, we'd segfault. This is such an incredible amount of trust to give to the functions you're calling. If you get a pointer, always check for null.

I could have fixed these, but I tried to keep everything bite sized for the average reader. It already took me years (!) to write these posts because I had to decide what to keep and what to cut, using experience working with a few real compilers in my job and class to help with that. There are plenty of things that discuss every little detail or have extremely well written, correct code. Those are generally textbooks.

But, thank you so much for reading! Feel free to contact me and get in touch if you have questions or concerns. I hope that compilers seem a little more like regular code now.
