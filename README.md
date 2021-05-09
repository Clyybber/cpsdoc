
# CPS - also known as Continuation-Passing Style - for Nim.

People watching the Nim community likely have heard the term "CPS" over the
last few months. It seems that there is confusion about what CPS _is_, what it
_is not_, what it can _do_ and what it can _not_ do.

This is a little writeup where we try to answer the above questions. It will
also go into some of the technicalities involved with CPS, and how this could
fit in the Nim ecosystem.

In this document I will use the term **CPS** for the programming style using
continuations, and **Nim-CPS** when referring to the particular implementation
that is currently under development.

## TL;DR

(If you have only 1 minute to spare, read this section only and ignore the rest
of this document)

*Nim-CPS* is a small library that can do only one simple thing: It basically
performs a transformation on Nim functions:

- It cuts one big "linear" function into smaller functions.
- It moves the local variables of the function out of the stack to a different
  place.

The end result of this transformation is a list of **continuations**, where a
continuation is typically a Nim object with a function pointer, and a list of
the variables that where originally on the stack. After this transformation,
the stack is no longer needed, which allows for some interesting possibilities:

- The transformed function can be interrupted half way, and resumed at a later
  time.  In the mean time one can run other functions (coroutines! iterators!)
  or one can wait for I/O to become available (async I/O!)

- The transformed parts of the functions can be called in a different order
  then intended, allowing you to build custom flow control primitives (goto!
  exceptions!) _without needing support from the language_.

- Parts of a function that are known to block (calculations, DNS lookups, etc)
  could be moved to another thread while keeping the main program responsive.
  (TODO What's this called)

- [TODO talk about Nim's bad threading support]

*Nim-CPS is not*:

- an alternative implementation of `async`. As a matter of fact, Nim-CPS does
not know anything about sockets, I/O, system calls, etc. That said, it is
perfectly possible to implement something like `async` using CPS as a building
block

- [TODO What more is Nim-CPS not]



# A little history of CPS

First, CPS is nothing new; in the Lisp world people have been doing CPS since
the '70s, and in some languages (mainly those of the functional-programming
style) CPS is a common programming paradigm.

Computer programs written in *procedural* programming languages are typically
composed of functions calling other functions, making up a kind of tree-shaped
form.

As an example of this control-flow, consider the following Nim program:

```nim
proc func3() =
  echo "three"

proc func2() =
  echo "two"
  func3()

proc func1() =
  echo "one"

proc main() =
  echo "entry"
  func1()
  func2()

main()
```

This is a rendering of that program's control-flow to show the tree structure:

```
 ----[main..] - - [..main..] - - - - - - - - - - [..main]---> end
            |     ^        |                     ^
            v     |        v                     |
            [func1] - - -  [func2..] - - [..func2]
                                   |     ^
                                   v     |
                                   [func3]
```

As control-flow is introduced, the tree -- also known as a **stack** -- grows
and shrinks; it must do so in order to keep track of variables the programmer
defined and the functions the programmer called so that it can *resume*
control-flow when those functions complete.

When a function *calls* another function, it will store the *return address* on
the stack; when the *called* function is done, the program will continue the
*calling* function from that return address.

This way of programming is structured and easy to reason about, but the
consequence is that there is only "one way" your program can flow: it goes into
functions, and only leaves them when they return. When your function calls
another function, the stack grows, consuming a limited memory resource that is
often inaccessible and of little use to the new function context.

If your program decides that it doesn't need some memory it consumed on the
stack in a parent function context, there's no easy way to recover that
resource. You might imagine that this is particularly problematic with
recursive functions, and you have perhaps run into a *stack overflow* error in
these cases.

A different approach to control-flow is called **CPS**. CPS is an acronym
for "Continuation-Passing Style", which sounds a bit abstract if you're not
knee-deep in computer science, but the concept is actually very simple and as
the name implies, it is merely a *style* of programming control-flow which you
can trivially adopt in almost any program.

Using CPS, when a function completes, it will never resume its caller using the
return address on the stack; instead it will directly call another function as
the last thing it does.

Let's rewrite our example in CPS.

```nim
proc func3() =
  echo "three"

proc func2() =
  echo "two"
  func3()

proc func1() =
  echo "one"
  func2()

proc main() =
  echo "entry"
  func1()

main()
```

Now let's see what our tree looks like.

```
 ----[main]                             [main] ---> end
          |                             ^
          v                             |
          [func1]                 [func1]
                |                 ^
                v                 |
                [func2]     [func2]
                      |     ^
                      v     |
                      [func3]
```

Hmm, not very compelling so far. The problem is that even though there's
nothing left to do at the end of our functions but to call another function,
the language is still growing the stack and never shrinking it until the last
function completes, at which point all the subsequent stack growth can be
unwound.

## A little history of Nim-CPS

Somewhere in the summer of '20, a few people started to think about what an
implementation of CPS could bring to Nim, and what this would look like. First
inspiration for in implementation was suggested by Araq, who pointed to a paper
describing a rudimentary CPS implementation for the C language.

Using the algorithms in this paper as a guide, a first implementation of
Nim-CPS was built using Nim macros, allowing CPS to be available as a library
only, without needing any support from the compiler or the language.


## What Nim-CPS can do for you, today

[TODO take a consice exmaple like stash/iterator, show what goes in and what comes out]


## The future of Nim-CPS

[TODO]



[TODO link to CPCS papers]

