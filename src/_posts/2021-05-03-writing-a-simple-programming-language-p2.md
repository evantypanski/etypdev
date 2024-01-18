---
layout: barebones
title: Writing a Simple Programming Language from Scratch - Part 2
date: 2021-05-03
permalink: /posts/writing-a-simple-programming-language-p2/
---

[Part 1 found here!!](https://etyp.dev/posts/writing-a-simple-programming-language-p1/)

Source code found [here](https://github.com/evantypanski/ectlang), for much easier running and compiling.

Up to this point, we made part of a compiler front end, namely the lexer and parser. Now, we'll adapt the parser and add some steps to make.. more code! Also known as an intermediate representation of the source code. More on that later.

To do this, we'll use a visitor pattern.

## Visitors
The [visitor software pattern](https://en.wikipedia.org/wiki/Visitor_pattern) is very useful, especially in compiler design. Visitors can essentially implement new methods on each node while traversing a tree. Or a not tree, but we'll use a tree.

Let's make a basic visitor!

> nodes.cpp
{:.filename}
``` cpp
#include <iostream>

class Node {
public:
    virtual void accept(class Visitor &v) = 0;
};


class IntegerNode: public Node {
public:
    IntegerNode(int value);
    void accept(Visitor &v);
    int value;
};

class Visitor {
public:
    void visit(IntegerNode *node);
};

IntegerNode::IntegerNode(int value) : value(value) {}

void IntegerNode::accept(Visitor &v) {
    v.visit(this);
}

void Visitor::visit(IntegerNode *node) {
    std::cout << node->value << std::endl;
}

int main() {
    IntegerNode node(5);
    Visitor visitor;
    node.accept(visitor);
}
```

Then here's the output when this program is called:

>
{:.shell}
``` bash
evan:ectlang/ $ g++ nodes.cpp
evan:ectlang/ $ ./a.out
5
evan:ectlang/ $ 
```

What's happening here is pretty simple. The node "accepts" the visitor, which prompts the visitor to visit that particular node. `accept` will call `IntegerNode`'s accept because of the `virtual` method. Then, the visitor can do whatever it wants with that object, knowing its type exactly. If we had a `PlusNode` that accepted a visitor, then we would call that accept method, thus we would visit the `PlusNode` instead.

I won't bore by going through all of the nodes, it's mostly just busy work to set it all up. There are ways to write those classes without all of the busy work (tools, metaprogramming). Useful, but writing a simple set of nodes isn't too much work. Sometimes people go through a lot of effort to automate a process that would take less time to just do manually.

So now we can add any number of nodes that we want. I'll show how we could do some recursion in this using the plus node, then hopefully extrapolating out will be a bit more simple.

> nodes.cpp
{:.filename}
``` cpp
/* Keeping all IntegerNode references in code */

class PlusNode: public Node {
public:
    PlusNode(IntegerNode left, IntegerNode right);
    void accept(Visitor &v);
    IntegerNode left, right;
};

class Visitor {
public:
    void visit(PlusNode *node);
};

PlusNode::PlusNode(IntegerNode left, IntegerNode right) :
    left(left), right(right) {}

void PlusNode::accept(Visitor &v) {
    v.visit(this);
}

void Visitor::visit(PlusNode *node) {
    node->left.accept(*this);
    std::cout << "+";
    node->right.accept(*this);
    std::cout << "=" << node->left.value + node->right.value << std::endl;
}


int main() {
    IntegerNode leftInt(1), rightInt(2);
    PlusNode plus(leftInt, rightInt);
    Visitor visitor;
    plus.accept(visitor);
}
```

This will recursively use your new `PlusNode` and output the added numbers! Here's the output:

>
{:.shell}
``` bash
evan:ectlang/ $ g++ nodes.cpp
evan:ectlang/ $ ./a.out
1+2=3
evan:ectlang/ $ 
```

The only nodes we'll add that aren't obvious are `ProgramNode` and `StatementNode`. A "program" will be a collection of statements for our simple language, and a "statement" is a bunch of math followed by a semicolon. Really a statement can be seen as a command, like `return 0;`

So how will we use this? The program can actually be represented easily as a tree. This is often represented as an abstract syntax tree (AST). We will then visit each node in the AST, and the visitor will generate some code. This generated code is an *intermediate representation* of the code - we don't write this code by hand, but the computer also doesn't use it. It's mainly used by the language to facilitate optimizations and compiling for different computers with different architectures.

We can also have other kinds of visitors! One class `Visitor` could be inherited by your type checker, or what makes your symbol table, then each of these visitors can be run. This example will only use one visitor - the one that generates code.

### Lexer and Parser Updates
Now let's update our parser from part 1 to return nodes instead of doing the calculations itself. We'll also rename the file the `parser.ypp` to generate C++. Here is what we'll use for our C declarations:

> parser.ypp
{:.filename}
``` cpp
%{
  #include <assert>
  #include <iostream>
  #include "nodes/node.h"
  #include "nodes/minusnode.h"
  #include "nodes/plusnode.h"
  #include "nodes/multnode.h"
  #include "nodes/divnode.h"
  #include "nodes/expnode.h"
  #include "nodes/integernode.h"
  #include "nodes/floatnode.h"
  #include "nodes/programnode.h"
  #include "nodes/statementnode.h"
  #include "scanner.c"

  using namespace std;
  extern "C" int yylex(void);
  extern int yylineno;

  void yyerror(char *s) {
    cerr << s << endl;
  }

  ProgramNode *program;
%}
```

There are some basic changes here, but two main ones for understanding how this works: first, we import all of the nodes we'll use inside the parser. Second, we declare a variable `*program` to point to the initial node. We'll build everything off of this, then use it in a different file.

Here are the Bison declarations that we'll use. We changed the union to include nodes that we want to be exchanged between grammar rules, and put the return values in their type definitions.

> parser.ypp
{:.filename}
``` cpp
%union{
  int intVal;
  float floatVal;
  class Node *node;
  class ExpNode *expNode;
  class StatementNode *statementNode;
}

%start program

%token <intVal> INTEGER_LITERAL
%token <floatVal> FLOAT_LITERAL
%token SEMI
%type <expNode> exp
%type <statementNode> statement
%type <node> program
%left	PLUS MINUS
%left	MULT DIV
```

This is similar to what we did in part 1 - just including the nodes!

Finally, here are the grammar rules:

> parser.ypp
{:.filename}
``` cpp
program: { program = new ProgramNode(yylineno); }
    | program statement	{ assert(program); program->addStatement($2); }
    ;

statement: exp SEMI { $$ = new StatementNode(yylineno, $1); }
    ;

exp:
   INTEGER_LITERAL  { $$ = new IntegerNode(yylineno, $1); }
    | FLOAT_LITERAL { $$ = new FloatNode(yylineno, $1); }
    | exp PLUS exp  { $$ = new PlusNode(yylineno, $1, $3); }
    | exp MINUS exp { $$ = new MinusNode(yylineno, $1, $3); }
    | exp MULT exp  { $$ = new MultNode(yylineno, $1, $3); }
    | exp DIV exp   { $$ = new DivNode(yylineno, $1, $3); }
    ;
```

Each rule returns a new node that we can access in another program as an object - after we finish parsing, we'll only touch these nodes, we don't care about the source code. So, when we parse a division, we'll create a `DivNode` to represent that division, then don't touch that source code again.

Now we also add a `yylineno` variable to keep track of line numbers. Most programming languages have some way of managing where in the source file certain nodes came from in order to emit diagnostics, telling the programmer what they did wrong. We're not going to do much with that, but if you ever want good diagnostics, consider including some source information.

Finally, `program` has a bit of interesting logic. If we get to the first, empty case for the `program` nonterminal, we make a new `ProgramNode` to use. If we get to the point where we have a statement, we'll also have a program, so we add the statement (where a `program` is just a `std::vector` of statements in implementation). We also assert if we don't have a `program` - some kind of `null` check is SO important in C++ code, preferably more than an `assert` since it doesn't apply in production! Never underestimate the power of humans to segfault a C/C++ program.

Our lexer didn't change much, just adding a couple options, but I'll put it here anyway:

> scanner.lex
{:.filename}
``` cpp
%{
#include "parser.tab.hpp"
  extern "C" int yylex();
%}
%option noyywrap
%option yylineno

%%
[0-9]+        { yylval.intVal = atoi(yytext); return INTEGER_LITERAL; }
[0-9]+.[0-9]+ { yylval.floatVal = atof(yytext); return FLOAT_LITERAL; }
"+"           { return PLUS; }
"-"           { return MINUS; }
"*"           { return MULT; }
"/"           { return DIV; }
";"           { return SEMI; }
[ \t\r\n\f]   ; /* ignore whitespace */
```

Now that we have that, let's make a file that runs this! This was moved from the bottom of the parser from part 1.

> main.cpp
{:.filename}
``` cpp
#include "nodes/programnode.h"
#include "nodes/visitor.h"

using namespace std;

extern ProgramNode* program;
extern int yyparse();

int main(int argc, char **argv) {
    yyparse();
    Visitor v;
    program->accept(v);
    return 0;
}
```

All this does is it calls some method `yyparse()` which we get as an extern function from the Bison files we generated. That populates our `program` variable. From this, we can visit the program we parsed!

Let's run it on a program with one line - `3 + 3.2;`

>
{:.shell}
``` bash
evan:ectlang/ $ flex -o scanner.c scanner.lex
evan:ectlang/ $ bison parser.ypp
evan:ectlang/ $ g++ parser.tab.cpp main.cpp nodes/*.cpp
evan:ectlang/ $ ./a.out < source.ect
Program
STATEMENT
3+3.2%
```

This is good! We visited the program, then the statement, then the `3+3.2` in our code. This proves we have a parser which creates the AST, then a visitor which goes through and visits the nodes.

## Code Generation

Now, we have an accurate AST for our program. We want to use this AST in order to write code that our computer can run. Normally, we'll do this with a true intermediate representation (like LLVM). We won't be doing that quite yet. Instead, we'll go with something easier to understand, that way the concept of code generation and visitors aren't blurred by the complexity of another foreign language. So, instead, we'll just turn our code into C++. It pains me to do this, but it's a good tool to use since we know what proper C++ code looks like.

Another consequence of writing C++ code is we're going to make it human readable. This won't be the case for most languages - once it gets to this point, the machine tends to be one of the only things actually reading the code, so it won't care much for readability. We're humans and C++ is a human programming language, so it's relatively easy to understand! LLVM is somewhat readable, but it's not super easy.

How will we do this? We'll make a new visitor that generates `.cpp` files! Here's the implementation file for that (let's ignore the header files):

> cppvisitor.cpp
{:.filename}
``` cpp
CPPVisitor::CPPVisitor(const char *filename) {
    // Open the file we are going to write to.
    file.open(filename, std::fstream::out);
}

void CPPVisitor::visit(ProgramNode *node) {
    // Write out the information we need to actually run the C program
    // This is hardcoded. It shouldn't be in most cases, but education!!
    file << "#include <iostream>\n";
    file << "int main() {\n";
    node->getStatement()->accept(*this);
    file << "}\n";
    file.close();
}

void CPPVisitor::visit(StatementNode *node) {
    // We want to print our statements when the generated C++ code is run.
    file << "\tstd::cout << ";
    node->getExp()->accept(*this);
    file << " << std::endl;\n";
}

void CPPVisitor::visit(IntegerNode *node) {
    // Append the integer to the file.
    file << node->getValue();
}

void CPPVisitor::visit(FloatNode *node) {
    // Append the float to the file.
    file << node->getValue();
}

void CPPVisitor::visit(PlusNode *node) {
    // Append the addition to the file.
    node->getLeft()->accept(*this);
    file << " + ";
    node->getRight()->accept(*this);
}

void CPPVisitor::visit(MinusNode *node) {
    // Append the subtraction to the file.
    node->getLeft()->accept(*this);
    file << " - ";
    node->getRight()->accept(*this);
}

void CPPVisitor::visit(MultNode *node) {
    // Append the multiplication to the file.
    node->getLeft()->accept(*this);
    file << " * ";
    node->getRight()->accept(*this);
}

void CPPVisitor::visit(DivNode *node) {
    // Append the division to the file.
    node->getLeft()->accept(*this);
    file << " / ";
    node->getRight()->accept(*this);
}
```

Basically, if you understood what was going on before with us printing out each statement, you'll understand this. Instead of using `std::cout` directly, we make a C++ program that calls them. Each statement (which, in our language, is just some math followed by a semicolon) will go into an `std::cout` call in another file - very, very useful! Okay maybe not but you can see the potential.

What's amazing about this isn't that we wrote a program that (kind of) writes C++ programs, it's that you can write your visitor to generate many different languages - it's not too much harder to just make it generate assembly! It's really easy to compile/run C++ code to see the fruits of your labor, so that's what we use here.

Let's run on this file (called `source.ect`):

>
{:.shell}
``` R
3 + 3.2;
5 + 2 * 3 - 8;
1.1 - 1.1 + 1.1 - 1.1 * 1;
```

One last run!

>
{:.shell}
``` bash
evan:ectlang/ $ flex -o scanner.c scanner.lex
evan:ectlang/ $ bison parser.ypp
evan:ectlang/ $ g++ parser.tab.cpp main.cpp nodes/*.cpp
evan:ectlang/ $ ./a.out < source.ect
```

From that, we get the following file created (in `test.cpp`):

> test.cpp
{:.filename}
``` cpp
#include <iostream>
int main() {
	std::cout << 3 + 3.2 << std::endl;
	std::cout << 5 + 2 * 3 - 8 << std::endl;
	std::cout << 1.1 - 1.1 + 1.1 - 1.1 * 1 << std::endl;
}
```

And if we run that...

>
{:.shell}
``` bash
evan:ectlang/ $ g++ test.cpp -o ect-program
evan:ectlang/ $ ./ect-program
6.2
3
0
```

Exactly as we'd expect. Good job, folks!

## So What?

So we created C++ code. Who cares? Why didn't we just write the C++ program? Well, the intermediate representation won't really be so high level - it'd be more like assembly in the end. We don't want to constantly write assembly for obvious reasons, so we write programs that transform our code into that - the whole purpose of a compiler. While here we made the intermediate representation a C++ file, that will never happen in a real language, but it's not too far off from it. That's why there's a bit more to the puzzle we have yet to cover.

With that being said, that's everything! You now should at least have a basis for doing some operations on your source code, eventually creating an intermediate representation that can be used. Congrats!

[Onwards to part 3!](https://dev.to/evantypanski/writing-a-simple-programming-language-from-scratch-part-3-1d7l)
