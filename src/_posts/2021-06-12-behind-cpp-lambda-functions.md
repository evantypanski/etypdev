---
layout: barebones
title: Behind C++ Lambda Functions
date: 2021-06-12
permalink: /posts/behind-cpp-lambda-functions/
---

Lambda functions are not just pointers to a function. This is obvious when you think about it: if they're anonymous, and declared completely differently, something must be different. But, we can assign a pointer to a function to a lambda easily:

``` cpp
void (*ptr)() = [](){};
```

So how does that work? We know we're just assigning to a pointer. But a lambda function isn't just a pointer. So what is it?

Note, we're going to use Clang as an example. A lot of things in C++ are up to compiler dev's imagination, also known as implementation specific. So if I say "lambdas have 12 toes," that is because Clang defines lambdas as having 12 toes, not because they have to have 12 toes.

## Lambdas are classes

When you define a lambda function, you're really making a C++ class. This class implements the method `operator()`. When you call the lambda, you're just calling that operator implementation.

Here's a simple C++ program to show that:

``` cpp
int main() {
  [](){}();
}
```

That line in main is just defining a lambda with no captures (`[]`), that takes no argument (`()`), with no body (`{}`), then calling it (`()`). We can actually look at what Clang develops as the AST with:

``` bash
clang++ -Xclang -ast-dump lambda.cpp
```

We'll walk through the lambda function piece by piece.

## Looking at the AST

The AST is just the internal representation of the code that the compiler looks at. Let's look at the most obvious part of the Lambda shenanigans (note, obvious is relative):

``` bash
`-CXXOperatorCallExpr 0x17238c8 <col:3, col:10> 'void':'void'
  |-ImplicitCastExpr 0x1723858 <col:9, col:10> 'void (*)() const' <FunctionToPointerDecay>
  | `-DeclRefExpr 0x17237d8 <col:9, col:10> 'void () const' lvalue CXXMethod 0x1723190 'operator()' 'void () const'
  `-ImplicitCastExpr 0x17238b0 <col:3, col:8> 'const (lambda at lambda.cpp:2:3)' lvalue <NoOp>
    `-MaterializeTemporaryExpr 0x1723898 <col:3, col:8> '(lambda at lambda.cpp:2:3)' lvalue
      `-LambdaExpr 0x17236a0 <col:3, col:8> '(lambda at lambda.cpp:2:3)'
```

Alright. What we have here is a call expression - that's the last `()` in our lambda. Then, we have some gibberish that is Clang doing internal conversions in order to make the types right. Finally, we have our trusted `LambdaExpr` - our lambda! Let's look at what's in that.

``` bash
`-LambdaExpr 0x17236a0 <col:3, col:8> '(lambda at lambda.cpp:2:3)'
  |-CXXRecordDecl 0x1723050 <col:3> col:3 implicit class definition
  | |-DefinitionData lambda pass_in_registers empty standard_layout trivially_copyable can_const_default_init
  | | |-DefaultConstructor defaulted_is_constexpr
  | | |-CopyConstructor simple trivial has_const_param needs_implicit implicit_has_const_param
  | | |-MoveConstructor exists simple trivial needs_implicit
  | | |-CopyAssignment trivial has_const_param needs_implicit implicit_has_const_param
  | | |-MoveAssignment
  | | `-Destructor simple irrelevant trivial
  | |-CXXMethodDecl 0x1723190 <col:6, col:8> col:3 used operator() 'void () const' inline
  | | `-CompoundStmt 0x1723240 <col:7, col:8>
  | |-CXXConversionDecl 0x1723538 <col:3, col:8> col:3 implicit operator void (*)() 'void (*() const noexcept)()' inline
  | |-CXXMethodDecl 0x17235e8 <col:3, col:8> col:3 implicit __invoke 'void ()' static inline
  | `-CXXDestructorDecl 0x17236d0 <col:3> col:3 implicit referenced ~ 'void () noexcept' inline default trivial
  `-CompoundStmt 0x1723240 <col:7, col:8>
```

Oh, gosh. We'll go piece by piece through this, but here's our class. [A `CXXRecordDecl` in Clang](https://clang.llvm.org/doxygen/classclang_1_1CXXRecordDecl.html) represents a C++ struct, union, or class. So our lambda actually contains that class in its node.

We can even see that in Clang documentation. If we go to the [documentation for `LambdaExpr` in Clang](https://clang.llvm.org/doxygen/classclang_1_1LambdaExpr.html), we can actually see a bunch of useful functions. Notably, we can see `getLambdaClass()`, which will "retrieve the class that corresponds to the lambda." Neat!

We'll skip the `DefinitionData` part of the class, since that's just defining some default constructors and whatnot. Let's look at the `operator()` definition:

``` bash
| |-CXXMethodDecl 0x1723190 <col:6, col:8> col:3 used operator() 'void () const' inline
| | `-CompoundStmt 0x1723240 <col:7, col:8>
```

Here we have our call operator. It's empty, and returns void, so it's not like we expect to see much. But, let's make it have something. If we change our lambda function to:

``` cpp
[](){int i = 1;}();
```

We now have something in our function body, so we expect to see it in the call operator:

``` bash
| |-CXXMethodDecl 0x171f190 <col:6, col:18> col:3 used operator() 'void () const' inline
| | `-CompoundStmt 0x171f2f8 <col:7, col:18>
| |   `-DeclStmt 0x171f2e0 <col:8, col:17>
| |     `-VarDecl 0x171f258 <col:8, col:16> col:12 i 'int' cinit
| |       `-IntegerLiteral 0x171f2c0 <col:16> 'int' 1
```

And there we have it, our call operator has a body, just nested in the AST. How about a return type?

``` cpp
[](){return 1;}();
```

In the AST:

``` bash
| |-CXXMethodDecl 0x1f88190 <col:6, col:17> col:3 used operator() 'int () const' inline
| | `-CompoundStmt 0x1f883b8 <col:7, col:17>
| |   `-ReturnStmt 0x1f883a8 <col:8, col:15>
| |     `-IntegerLiteral 0x1f88240 <col:15> 'int' 1
```

As we can see, our `CXXMethodDecl` changed from `void () const` to `int () const`, showing our change in return type!

Finally, how about adding a parameter?

``` cpp
[](int i){}(1);
```

AST:

``` bash
| |-CXXMethodDecl 0xe84220 <col:11, col:13> col:3 used operator() 'void (int) const' inline
| | |-ParmVarDecl 0xe83fc8 <col:6, col:10> col:10 i 'int'
| | `-CompoundStmt 0xe842d8 <col:12, col:13>
```

We can see that the `CXXMethodDecl` is now `void (int) const` and we have a `ParmVarDecl` (just a function parameter) inside of the method decl. Awesome.

But, lambdas also let you capture a variable. This means that if I have a variable `i` outside of the lambda, capture it, change it, then call the lambda, we should see that variable change. Let's try it out:

``` cpp
#include <iostream>

int main() {
  int i;
  auto lambda = [&i](){
    std::cout << "i is: " << i << std::endl;
  };
  i = 42;
  lambda();
}
```

And here's our output:

``` bash
i is: 42
```

Awesome. How does this work in the AST? If we look at it, we actually have one key difference:

``` bash
| |-FieldDecl 0x18ec440 <col:5> col:5 implicit 'int &'
```

We have a field inside of our lambda's class now! It's a reference to an int, namely to `i` declared outside. Then, when we use our lambda, we just use that field! Neat.

Now, we don't take a parameter, we still return void, can we assign this to a C function pointer like we did at the beginning?

``` cpp
int main() {
  int i;
  void (*ptr)() = [&i](){};
}
```

The compiler errors!

``` bash
lambda.cpp:5:10: error: no viable conversion from '(lambda at lambda.cpp:5:19)' to 'void (*)()'
  void (*ptr)() = [&i](){};
         ^        ~~~~~~~~
```

It's weird because we still have all the qualifications of that function pointer. But, if we look at the AST, we'll see something missing from the Lambda's class, particularly:

``` bash
| |-CXXConversionDecl 0x142f7b8 <col:19> col:19 implicit used operator void (*)() 'void (*() const noexcept)()' inline
| | `-CompoundStmt 0x142fba0 <col:19>
| |   `-ReturnStmt 0x142fb90 <col:19>
| |     `-ImplicitCastExpr 0x142fb78 <col:19> 'void (*)()' <FunctionToPointerDecay>
| |       `-DeclRefExpr 0x142fb58 <col:19> 'void ()' lvalue CXXMethod 0x142f868 '__invoke' 'void ()'
```

Clang actually refuses to generate a conversion to the function pointer if we have any captures! That makes sense because the function now relies on that variable `i`.

Let's say we're really curious and need to know why this happened. How can we do that?

## Diving into Clang's code

Luckily, Clang's source code is open source. You can find it, along with LLVM and more, [on Github](https://github.com/llvm/llvm-project). Furthermore, if you want to know something high level about Clang's code, check out the [Clang internals manual](https://clang.llvm.org/docs/InternalsManual.html).

But, our job is simple: find out where it refuses to generate the conversion function.

The first place to start requires a bit of compiler knowledge. Clang has a Sema library for certain semantic actions. This Sema library is what builds the AST. Considering that, we probably have a good place to start, since we noticed a difference in the AST.

At this time, you can find the Sema library on the Github under `llvm-project/clang/lib/Sema/`

Now that we're here, we want to see what creates the lambda function. A solid way to do this would be to look up the constructor in the documentation and see what calls it and when. But, our case is lucky: there's a file called `SemaLambda.cpp`, so let's check that out.

If we dig around here, we find the function `BuildLambdaExpr`. Seems promising.

From there, let's just look through the function.

First we do some setup from previously gathered info:
``` cpp
CallOperator = LSI->CallOperator;
Class = LSI->Lambda;
IntroducerRange = LSI->IntroducerRange;
ExplicitParams = LSI->ExplicitParams;
ExplicitResultType = !LSI->HasImplicitReturnType;
LambdaCleanup = LSI->Cleanup;
ContainsUnexpandedParameterPack = LSI->ContainsUnexpandedParameterPack;
IsGenericLambda = Class->isGenericLambda();
```

Then, we do some logic checking if captures are used and getting them in a good form for our AST.

Next, we can see why we don't get our conversion function:

``` cpp
if (Captures.empty() && CaptureDefault == LCD_None)
  addFunctionPointerConversions(*this, IntroducerRange, Class,
                                CallOperator);
```

So, we only add the function pointer conversion if we have no captures and the capture default is none!

If you really want to dive into the weeds, the Clang devs also often comment the part of the standard they took from:

``` cpp
// C++11 [expr.prim.lambda]p6:
//   The closure type for a lambda-expression with no lambda-capture
//   has a public non-virtual non-explicit const conversion function
//   to pointer to function having the same parameter and return
//   types as the closure type's function call operator.
```

You can find this yourself too! Just look around for C++ standards (like through [open-std.org](http://www.open-std.org/JTC1/SC22/WG21/docs/papers/) or just a search engine) and find the section with expr.prim.lambda. You'll see numbering, find number 6, and you'll find identical or similar wording to that.

## Was this about lambdas?

Not really. Lambdas were just a mechanism for showing how to find out more about C++ by digging into it. This applies to any language or compiler too, especially if they let you dump ast (or some intermediate representation). It's also just fun to look at complex code.

So there you go. Try this out the next time you see a weird construct in your favorite language. Sometimes, you'll get surprised and fall into a fun rabbit hole. It's worth every hour.
