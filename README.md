
# CPS - also known as Continuation-Passing Style - for Nim.

People watching the Nim community likely have heard the term "CPS" over the
last months. It seems that there is a lot of confusion about what CPS _is_, what
it _is not_, what it can _do_ and what it can _not_ do.

This is a little writeup where we try to answer the above questions. It will
also go into some of the technicalities involved with CPS, and how this could
fit in the Nim ecosystem.

In this document I will use the term "CPS" for the programming style using
continuations, and "Nim-CPS" when referring to the particular implementation
that is currently under development.

## TL;DR

(If you have only 1 minutes to spare, read this section only and ignore the rest
of this document)

*Nim-CPS is* a small library that can do only one simple thing: It basically
performs a transformation on Nim functions:

- It cuts one big "linear" function into smaller functions.
- It moves the local variables of the function out of the stack to a different
  place.

The end result of this transformation is a list of "continuations", where a
continuation is typically a Nim object with a function pointer, and a list of
the variables that where originally on the stack. After this transformation,
the stack is no longer needed, which allows for some interesting possibilities:

- The transformed function can be interrupted half way, and resumed at a later
  time.  In the mean time one can run other functions (coroutines! iterators!)
  or one can wait for I/O to become available (async I/O!)

- The transformed parts of the functions can be called in a different order
  then intended, allowing you to build custom flow control primitives (goto!
  exceptions!) _without needing support from the language_.

- Parts of a function that are know to block (calculations, DNS lookups, etc)
  could be moved to another thread while keeping the main program responsive.
  (TODO What's this called)

- [TODO talk about Nim's bad threading support]

*Nim-CPS is not*:

- an alternative implementation of `async`. As a matter of fact Nim-CPS does not
  know anything about sockets, I/O, system calls, etc. That said, it is perfectly
  possible to implement something like `async` using CPS as a building block

- [TODO What more is Nim-CPS not]



# A little history of CPS

First things first: CPS is nothing new; in the Lisp-world people have been
doing CPS since the 70's, and in some languages (mainly functional-programming
style) CPS is a common programming paradigm.

"Traditional" computer programs are typically composed of functions calling other
functions, making up a kind of tree-shaped form. Every function will in the end
return, until the program is done. When a function calls another function, it
will store the return address on the stack; when the new function is done, the
computer will continue the previous function from that address.

```
 ----[main..] - - [..main..] - - - - - - - - - - [..main]---> end
            |     ^        |                     ^
            v     |        v                     |
            [func1] - - -  [func2..] - - [..func2]
                                   |     ^ 
                                   v     | 
                                   [func3] 
```

[TODO little drawing of program flow in a tree]

This way of programming is nice and structured, but the consequence is that
there is only "one way" your code can flow: it goes in functions, and only out
when they return.

There are different ways of writing software, one of which is called CPS.  CPS
is an acronym for "Continuation-passing style", which sounds a bit abstract if
you're not knee deep into computer science. What happens is this:

When a function is done, it will never return back to the place where it came
from; instead it will directly call another function as the last thing it does.

```
 ----[main]   
          |   
          v    
          [func1]
                |   
                v    
                [func2]
                      |   
                      v    
                      [func3]
                            |   
                            v    
                            [func3] ---> end
                 
```

[TODO tell about CTO?]


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

