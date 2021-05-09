
# CPS - also known as Continuation-Passing Style - for Nim.

People watching the Nim community have likely heard the term "CPS" over the
last few months. It seems there is some confusion about CPS. Specifically, what
CPS _is_, what it _is not_, what it can _do_ and what it can _not do_.

This short writeup attempts to answer the above questions and describes a few
technicalities involved with CPS, and how this could fit in the Nim ecosystem.

NB: This document uses the term **CPS** for the programming style using
continuations, and **Nim-CPS** when referring to the particular implementation
that is currently under development.

## TL;DR

(If you have only 1 minute to spare, read this section)

*Nim-CPS* is a small metaprogramming library that offers a only a simple
transformation for Nim functions. This transformation:

- Cuts a function into smaller functions. Cuts are made at points where control-flow changes, e.g. `if`, `return`, calls, etc.
- Moves local variables of the function from the stack to the heap.

The end result of this transformation is a continuation type, which is
typically a Nim object which contains a function pointer and any variables
that where originally on the stack in the original function. In addition, a
series of functions are generated which take this **continuation** as input and
perform any continuation-free logic from the body of the original function.

After this transformation, the stack is no longer needed, which allows for some
interesting possibilities:

- Pre-transformed functions _may_ be interrupted part-way at each cut point.
- The transformed parts of a function _may_ be called in a different order,
  or not called at all.
- Some or all of the transformed parts _may_ be run in different threads.

Note that _may_ was often used, this is where other libraries come in to use
Nthe im-CPS transform in order to implement:

- Interrupting and resuming function execution part-way, while running another
  function's parts is the basis of `yield` (coroutines! iterators! generators!),
  or one can wait for I/O to become available (async I/O!).

- Varying the order of calling the transformed parts than originally intended
  allows you to build custom flow control primitives (goto! exceptions!)
  _without needing support from the language_.

- Moving parts of a function that are known to block (calculations, DNS lookups,
  etc) to another thread while keeping the main program responsive (background
  processing!).

- [TODO talk about Nim's threading support]

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

## Control-flow on the Stack

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
  echo "exit"

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

## Stack Growth

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

## Enter CPS

A different approach to control-flow is called **CPS**. CPS is an acronym
for "Continuation-Passing Style", which sounds a bit abstract if you're not
knee-deep in computer science, but the concept is actually very simple and as
the name implies, it is merely a *style* of programming control-flow which you
can trivially adopt in almost any program.

Using CPS, when a function completes, it will never resume its caller using the
return address on the stack; instead it will directly call another function as
the last thing it does.

## Simplifying Control-flow with CPS

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
  echo "exit"

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

## There's Just One Problem

This is silly! After leaving each of these functions, `main`, `func1`, `func2`,
and even `func3`, we never return to the caller's function context, yet we
still pay the penalty of having visited in the first place.

However, when you think about it, the program is already simpler to follow and
we've managed to constrain this problem of optimizing or eliminating stack
growth to a single scenario: a function call at the tail of another function.

## Tail-Call Optimization

Indeed, `gcc` and `clang` may be able to recognize that control-flow need
never revisit the calling function and thus unwind the stack before making
the function call at the tail. This is appropriately-termed **Tail-Call
Optimization** or **Tail-Call Elimination**.

Unfortunately, this optimization is not guaranteed by Nim, and as a result, we
need to address stack growth directly in Nim.

## Enter the Trampoline

What we really want is to run the first function, then run the next function,
then run the next function, and so on until there are no more functions to run.

To achieve this, we simply have each function in the chain tell the program
where to go next.

```nim
proc done(): auto =
  return done

proc func3(): auto =
  echo "three"
  return done

proc func2(): auto =
  echo "two"
  return func3

proc func1(): auto =
  echo "one"
  return func2

proc main() =
  echo "entry"

  # start with a continuing function
  var fn = func1

  # bounce on the trampoline until the continuation is done
  while fn != done:
    # run the current function and replace it
    # with whatever the next function is
    fn = fn()

  echo "exit"

main()
```

Now our stack tree looks like this:

```
 ----[main]     [main]     [main]     [main] ---> end
          |     ^    |     ^    |     ^
          v     |    v     |    v     |
          [func1]    [func2]    [func3]
```

Perfect! We performed the same control-flow, more simply, and with no
inefficient stack growth.

## You Call That 'Simple'?

Hang in there; it's going to get worse before it gets better.  😁

Let's modify our original program to add some more complexity.

```nim
import times

proc okay() =
  echo "okay"

proc spin() =
  if getTime().toUnix mod 2 == 0:
    echo "come back later"
    spin()
  else:
    okay()

proc main() =
  echo "entry"
  spin()
  echo "exit"

main()
```

Now the program prints `come back later` until the epoch time is odd, at
which point it prints `okay` and then completes, printing `exit`. This is a
very elegant control-flow design, but there's a bug in this program because
sometimes we spin so much that we exhaust the stack just by bookkeeping the
entry and exit of the `spin` procedure.

Let's see how this is solved in CPS.

```nim
import times

proc okay(): auto =
  echo "okay"
  okay

proc spin(): auto =
  if getTime().toUnix mod 2 == 0:
    echo "come back later"
    spin
  else:
    okay

proc main() =
  echo "entry"
  var fn = spin
  while fn != okay:
    fn = fn()
  echo "exit"

main()
```

Actually, that wasn't so bad. In fact, it's almost the same size as the
original version, but this one doesn't have the original's stack-overflow bug.

# Nim-CPS

## A little history of Nim-CPS

Somewhere in the summer of '20, a few people started to think about what an
implementation of CPS could bring to Nim, and what this would look like. First
inspiration for in implementation was suggested by @Araq, who pointed to a
paper describing a rudimentary CPS implementation for the C language.

Using the algorithms in this paper as a guide, a first implementation of
Nim-CPS was built using Nim macros, allowing CPS to be available as a library
only, without needing any support from the compiler or the language.


## What Nim-CPS can do for you, today

[TODO take a concise example like stash/iterator, show what goes in and what comes out]


## The future of Nim-CPS

[TODO]



[TODO link to CPCS papers]

