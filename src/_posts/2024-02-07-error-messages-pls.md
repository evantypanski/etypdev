---
layout: barebones
title: The Anatomy of an Error Message
date: 2024-02-06
permalink: /posts/errors-pls/
---

Programming languages are fun to design and think about. The error messages aren't as much. But let's look at some anyway. Don't worry, we'll look at compiler source code too :)

## Clang

First, let's say we just programmed a lot of Rust and forgot that C++ requires parentheses around the `if` condition:

> test.cpp
{:.filename}
``` cpp
int main() {
  if true {
    int i = 0;
  }
}
```

Clang reacts:

>
{:.shell}
``` bash
etyp:errors-pl/ $ clang++ test.cpp
test.cpp:2:6: error: expected '(' after 'if'
  if true {
     ^
1 error generated.
```

This is the quintessential compiler error. It doesn't necessarily tell you how to fix it unless you've been programming for a bit and understand this jargon. Why did you expect a `(`? What happens after that? What goes in there?

A lot of compilers do this just because it's how compilers work. Let's look at Clang's code to see why.

This error is from parsing. That means it's still checking the structure of the source code before figuring out what it means.

In C++, there are a few ways to write the first part of an `if` statement. We'll ignore the C++23 `consteval` if statements. That means the initial part can be written like:

>
{:.filename}
``` cpp
if (condition)
```

Or:

>
{:.filename}
``` cpp
if constexpr (condition)
```

Now, we won't go into what `constexpr` means here, but the important part is we have two choices: either we see a `constexpr` or not. In Clang's source, we can find the check for `constexpr` in the function that parses `if` statements (suitably named `ParseIfStatement`):

> ParseStmt.cpp
{:.filename}
``` cpp
if (Tok.is(tok::kw_constexpr)) {
  Diag(Tok, getLangOpts().CPlusPlus17 ? diag::warn_cxx14_compat_constexpr_if
                                      : diag::ext_constexpr_if);
  IsConstexpr = true;
  ConsumeToken();
}
```

At this point, we should always see a `(` (also known as left parentheses or lparen). So we check for that. Here's that check:

> ParseStmt.cpp
{:.filename}
``` cpp
if (!IsConsteval && (NotLocation.isValid() || Tok.isNot(tok::l_paren))) {
  Diag(Tok, diag::err_expected_lparen_after) << "if";
  SkipUntil(tok::semi);
  return StmtError();
}
```

Ok we ignored the `consteval` stuff so the only important part of that condition is `Tok.isNot(tok::l_paren)`. If our next token is not an lparen, then we throw an error. All of these errors are in a file called `DiagnosticParseKinds.td`, which generates C++ code for these. If we search for that error we find:

> DiagnosticParseKinds.td
{:.filename}
```
def err_expected_lparen_after : Error<"expected '(' after '%0'">;
```

Easy enough. That's the error we saw.

### Source snippets

The error message we saw contains a snippet of the code. How did we get that? Well, in Clang we can look back at the `Diag` call and see some arguments:

> ParseStmt.cpp
{:.filename}
``` cpp
Diag(Tok, diag::err_expected_lparen_after) << "if";
```

The `Tok` argument gets passed in! In the parser we have a function:

> Parser.cpp
{:.filename}
``` cpp
DiagnosticBuilder Parser::Diag(const Token &Tok, unsigned DiagID) {
  return Diag(Tok.getLocation(), DiagID);
}
```

... which takes in a token. Exactly what we wanted. That ends up calling an overloaded variant where it just takes the `getLocation`, which returns a [`SourceLocation`](https://clang.llvm.org/doxygen/classclang_1_1SourceLocation.html):

> Parser.cpp
{:.filename}
``` cpp
DiagnosticBuilder Parser::Diag(SourceLocation Loc, unsigned DiagID) {
  return Diags.Report(Loc, DiagID);
}
```

I don't really want to get into that `Report` function since it's pretty straightforward: We find the location and we print a format string that points to that location. But those source locations are a bit interesting.

Here's Clang's description of a source location:

> The SourceManager can decode this to get at the full include stack, line and column information.
> Technically, a source location is simply an offset into the manager's view of the input source, which is all input buffers (including macro expansions) concatenated in an effectively arbitrary order. The manager actually maintains two blocks of input buffers. One, starting at offset 0 and growing upwards, contains all buffers from this module. The other, starting at the highest possible offset and growing downwards, contains buffers of loaded modules.

Ok, so it's just an offset? It turns out, we can look at the class and see one singular field:

> Parser.h
{:.filename}
``` cpp
UIntTy ID = 0;
```

Yep, so we can get all of that info from one 32 bit field. This includes macro expansion shenanigans!

We won't get into macro shenanigans; don't worry. If you're not convinced it's hard, what if our snippet was this:

> test.cpp
{:.filename}
``` cpp
# define MY_IF if true

int main() {
  MY_IF {
    int i = 0;
  }
}
```

Where do we show the error, where the macro is in source, or in the macro? This isn't obvious since the error is actually inside the replacement from the macro. Well, it's both:

>
{:.shell}
``` bash
etyp:errors-pl/ $ ~/src/llvm-project/build-debug/bin/clang++ test.cpp
test.cpp:4:3: error: expected '(' after 'if'
    4 |   MY_IF {
      |   ^
test.cpp:1:19: note: expanded from macro 'MY_IF'
    1 | # define MY_IF if true
      |                   ^
1 error generated.
```

So... we need a way to unwrap this macro in order to actually give the user useful information. We'll just take a quick glance at code without dealing with the macro parts. This becomes a LOT when we have large nested macros.

Given a source location, then, what actually turns that offset into a snippet of source code to display? Well that's the [`SourceManager`](https://clang.llvm.org/doxygen/classclang_1_1SourceManager.html), of course!

So, there's actually a lot that goes into this. C++ is unlike a lot of programming languages, where it has these `#include` directives which just put another file into your file in its entirety (minus some preprocessor stuff or language extensions). We need to keep track of files and macros and it's all a pain. Because there's all that extra complexity, I'll just describe the implementation with pretty few source snippets.

First, we need the line number. That's important for telling you where the error is, right? Well, say we have a 100 character file and it's in an array, like `char MyFile[100]`. We want to find the line number for offset `42`. How can we do this?

The brute force method would be just go through `0` to `42` and count the number of newline characters. Easy enough, but that's not really efficient. Clang doesn't do this.

Instead Clang has a cache! We have a cache for the content in the file called [`ContentCache`](https://clang.llvm.org/doxygen/classclang_1_1SrcMgr_1_1ContentCache.html). That has its own cache, suitably named `SourceLineCache`. The description in the documentation is:

> A bump pointer allocated array of offsets for each source line.

Okay let's not get into all of that, but we store what positions hold newlines and we search for where our position falls in that. Then we get the line number.

Column number is actually pretty easy if we have the line number! Minus a bunch of caching stuff, here's the calculation:

> SourceManager.cpp
{:.filename}
``` cpp
return FilePos - LineStart + 1;
```

So if the source location is at offset `50`, the line starts at offset `20`, then we're at `50 - 20 + 1 = 31` (note `+1` to un-zero index it). So now we know where in the file it is!

Now with that info, it's pretty straightforward to display a snippet (again... ignoring macros and other complications). We have a cache telling us where line numbers are. So we need this line, which we have its start in that cache. Go to the previous element of that cache and we can get the previous line too, to give some context. Then just print that out to the console.

I know some of this was rushed over and there's less code examples here, but let's just say the extra complications from macros and different files actually makes it hard to show without polluting the simplified example. Sorry :(

### Fix it!

Okay, so at this point we have a whole system of reporting errors at the correct location. There's one thing that seems missing here: fixing it! In Clang's [diagnostics documentation](https://clang.llvm.org/diagnostics.html), they mention a couple cases where the user gets hints to fix the issue. Here's an example:

> test.cpp
{:.filename}
``` cpp
template<typename T> struct S {};

struct S<int> {};
```

>
{:.shell}
``` bash
etyp:errors-pl/ $ clang++ test.cpp
test.cpp:3:8: error: template specialization requires 'template<>'
    3 | struct S<int> {
      |        ^~~~~~
      | template<>
1 error generated.
```

Okay it's a little weird that it points to `S` even though you need to put the fixit hint before the `struct` keyword. According to the page I linked, here's the diagnostic:

>
{:.shell}
``` bash
$ clang t.cpp
t.cpp:9:3: error: template specialization requires 'template<>'
  struct iterator_traits<file_iterator> {
  ^
  template<>
```

... which puts `template<>` at the correct location. I won't harp on this too much, but one of the more frustrating experiences you can have is to do exactly what the compiler suggests and have that be wrong. This is most likely a result of making the columns more accurate for other parts and this particular case just never got caught.

Okay, so how does this fixit work? The error is:

> DiagnosticSemaKinds.td
{:.filename}
``` cpp
def err_template_spec_needs_header : Error<
  "template specialization requires 'template<>'">;
```

So it doesn't have any extra arguments for fixit, so it must be when it's used:

> SemaTemplate.cpp
{:.filename}
``` cpp
Diag(DeclLoc, diag::err_template_spec_needs_header)
  << Range
  << FixItHint::CreateInsertion(ExpectedTemplateLoc, "template<> ");
```

There we go, we have a class `FixItHint` that we can create a hint for the user. Let's try to use this for our simple case of missing parentheses.

Here's the line that diagnoses the parentheses error we saw before:

> ParseStmt.cpp
{:.filename}
``` cpp
Diag(Tok, diag::err_expected_lparen_after) << "if";
```

As an experiment, here's a first draft of adding the lparen:

> ParseStmt.cpp
{:.filename}
``` cpp
Diag(Tok, diag::err_expected_lparen_after)
  << "if"
  << FixItHint::CreateInsertion(Tok.getLocation(), "(");
```

Here's the new error I get:

>
{:.shell}
``` bash
etyp:errors-pl/ $ ~/src/llvm-project/build-debug/bin/clang++ test.cpp
test.cpp:2:6: error: expected '(' after 'if'
    2 |   if true {
      |      ^
      |      (
1 error generated.
```

That's actually relatively informative, I think! It shows what would be expected and it makes it slightly clearer what it means.

Now, I think one weird part here is that adding in the fix it hint would not create a compileable program. If you just blindly follow it and only add the lparen, you get:

>
{:.shell}
``` bash
etyp:errors-pl/ $ ~/src/llvm-project/build-debug/bin/clang++ test.cpp
test.cpp:2:12: error: expected ')'
    2 |   if (true {
      |            ^
test.cpp:2:6: note: to match this '('
    2 |   if (true {
      |      ^
test.cpp:5:1: error: expected statement
    5 | }
      | ^
2 errors generated.
```

We could go in and add another fixit, but it does have a note to help. I think the problem here is a bit deeper: the fixit hint doesn't completely fix the problem, it just provides a solution that needs an extra step. That's not much clearer in my opinion.

So what if we try to fix that? Well, it's a fair bit more complicated. We don't know what mistakes the user made beyond this point. Trying to give a good fixit here would require some assumptions. If we assume that the user did provide a `{` (lbrace) where it would be necessary, then we can just skip until we see the lbrace and suggest a fixit surrounding the stuff before in parentheses.

When you write a parser, you (should) try to make it recover really really well from errors like this so that the user doesn't have to continue to recompile just to get syntax right. But sometimes it's just hard to have something that covers every case. So instead, you get somewhat vague error messages. This case isn't exactly vague, but it is a specific type of jargon that is just another step to learning how to program.

## Other languages

My favorite language for error messages is Rust, and rightfully so. It's a very complicated language, so in order to get people to use it, diagnostics have to be precise and tell the user what to do.

Let's look at an analogous example with the `if` statement. Rust doesn't require parentheses around the condition, but requires what follows to be a block surrounded by `{` and `}`:

> test.rs
{:.filename}
``` rust
fn main() {
  if true
    let i = 0;
}
```

We get this error:

>
{:.shell}
``` bash
etyp:errors-pl/ $ rustc test.rs
error: expected `{`, found keyword `let`
 --> test.rs:3:5
  |
3 |     let i = 0;
  |     ^^^ expected `{`
  |
note: the `if` expression is missing a block after this condition
 --> test.rs:2:6
  |
2 |   if true
  |      ^^^^
help: try placing this code inside a block
  |
3 |     { let i = 0; }
  |     +            +

error: aborting due to previous error
```

I think that's awesome. We get the "compiler jargon" error, then a note saying WHY we got that. Then, because it's a common mistake, we also have a special case that gives an easy fix for our issue. A super simple case where almost any other language simply gives the first one, but the standard Rust compiler goes above and beyond.

Then just to add another language with a bad error message, here's what I get with my system installed golang:

> test.go
{:.filename}
``` go
package main

func main() {
  if true
    var _ = 0
}
```

>
{:.shell}
``` bash
etyp:errors-pl/ $ go run test.go
# command-line-arguments
./test.go:5:3: syntax error: unexpected var, expected expression
```

At least it gives the column number?

## Conclusions

Maybe I'm weird, but I find that the quality of error messages is proportional to how much I enjoy a language. It should tell me what I did wrong and why.

Clang's error messages are pretty good in the C/C++ world. For the opposite end of the spectrum, just take what `cl` (Microsoft's C/C++ compiler) does with the example we looked at:

>
{:.shell}
``` bash
example.cpp
<source>(2): error C2059: syntax error: 'constant'
<source>(2): error C2143: syntax error: missing ';' before '{'
Compiler returned: 2
```

I know how to program with C++, I know a fair bit about programming languages, and I've seen a lot of them. But I have no idea what those errors mean. Basically the only piece of information I get is that it's on line 2, and then some... less than helpful information.

So do better than MSVC. Your users deserve better. Oh yeah, this applies to websites too by the way... I just like compilers.
