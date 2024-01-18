---
layout: barebones
title: This C++ code doesn't look ambiguous to me
date: 2023-10-23
permalink: /posts/this-cpp-is-ambiguous/
---

Look at this C++ code:

> test.cpp
{:.filename}
``` cpp
class B;
class A { public: A (B&);};
class B { public: operator A(); };
class C { public: C (B&); };
void f(A) { }
void f(C) { }

int main() {
  B b;
  f(b);
}
```

Do you think this compiles? Well, probably not because I'm asking. So here's the error message from Clang 13:

>
{:.shell}
``` bash
etyp:fun/ $ clang++ test.cpp
test.cpp:10:3: error: call to 'f' is ambiguous
  f(b);
  ^
test.cpp:5:6: note: candidate function
void f(A) { }
     ^
test.cpp:6:6: note: candidate function
void f(C) { }
     ^
1 error generated.
```

Okay so it's ambiguous which conversion gets done for the object `b` (of type `B`). So what options are there?

1) Call `f(A)`. We can call `A`'s constructor which takes a reference to a `B`

2) Call `f(C)`. We can call `C`'s constructor which takes a reference to a `B`

3) Call `f(A)`. We can call `operator A()` in the class `B`

But, 1 and 3 are ambiguous because we can't find a better one to call `f(A)`. But `f(C)` is still not ambiguous - we know how to call it without any ambiguity. Trying to call `f(A)` is ambiguous so *obviously* it shouldn't be chosen.

It turns out in this case, the ambiguity between 1 and 3 is ranked equal priority as the user defined conversion in 2. From the C++20 standard section 12.4.2.10:

> For the purpose of ranking implicit conversion sequences as described in 12.4.3.2, the ambiguous conversion sequence is treated as a user-defined conversion sequence that is indistinguishable from any other user-defined conversion sequence.

I find this surprising because I could easily determine that option 2 is the unambiguous way to make this compile.

## Not all compilers...

It turns out, MSVC actually [accepts the code](https://godbolt.org/#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,selection:(endColumn:1,endLineNumber:12,positionColumn:1,positionLineNumber:12,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:'class+B%3B%0Aclass+A+%7B+public:+A+(B%26)%3B%7D%3B%0Aclass+B+%7B+public:+operator+A()%3B+%7D%3B%0Aclass+C+%7B+public:+C+(B%26)%3B+%7D%3B%0Avoid+f(A)+%7B+%7D%0Avoid+f(C)+%7B+%7D%0A%0Aint+main()+%7B%0A++B+b%3B%0A++f(b)%3B%0A%7D%0A'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0')),k:33.333333333333336,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:vcpp_v19_37_x64,deviceViewOpen:'1',filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,libs:!(),options:'',overrides:!(),selection:(endColumn:1,endLineNumber:1,positionColumn:1,positionLineNumber:1,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:1),l:'5',n:'0',o:'+x64+msvc+v19.37+(Editor+%231)',t:'0')),k:33.333333333333336,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:output,i:(compilerName:'x86-64+gcc+13.1',editorid:1,fontScale:14,fontUsePx:'0',j:1,wrap:'1'),l:'5',n:'0',o:'Output+of+x64+msvc+v19.37+(Compiler+%231)',t:'0')),k:33.33333333333333,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4). You can even see in the assembly it calls `f(C)`.

I believe no matter how silly the standard is with its rules (which this particular point may have a very good reason), C++ compilers should aim to conform to the standard. This case was pulled exactly from the standard as a case that should not compile. MSVC should not compile this.

But that's a whole other rant.

Thanks for looking at that C++ code.
