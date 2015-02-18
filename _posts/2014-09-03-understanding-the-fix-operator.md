---
layout: post
title: "Understanding the fix operator"
description: ""
category:
tags: []
---
{% include JB/setup %}

I finally got some intuition for the [fixed-point combinator](http://en.wikipedia.org/wiki/Fixed-point_combinator).
To be more precise, I got an intuition for why it exists, not why it's defined as it is. Here goes:

Recursion in Lambda-Calculus
----------------------------

Let us try to write a simple recursive function, factorial, in the lambda-calculus with some extensions
for integers and conditionals. Because I know how to write it in Scala, here is a Scala implementation:

{% highlight scala %}
def factorial(n: Int): Int = if (n == 0) 1 else n * factorial(n - 1)
{% endhighlight %}

Here's a first attempt in lambda-calculus:

    factorial = λn. if iszero n then 0 else factorial ...

But wait! We cannot re-use `factorial`: it is a *meta-variable*, and has no meaning in the host language. If
we want to use a value inside an expression, it has to be abstracted over:

    g = λfactorial. λn. if iszero n then 0 else factorial (n - 1)

What is going on now? Let us look at the types of `g`, `factorial`. Let us write g in the
simply typed lambda-calculus:

    g = λfactorial: Int => Int. λn: Int. if iszero n then 0 else factorial (n - 1)

This is good, because the factorial function has the type we expect for it. The type of `g`, though,
is `(Int => Int) => Int => Int`: it takes a function type, an integer type, and returns an integer.
We still want a function of the type `Int => Int`, the original expected type of the factorial
function. How do we achieve that?

Let us rewrite the type of `g`, and re-interpret it: `(Int => Int) => (Int => Int)`. I have just
added unnecessary brackets. But this changes the interpretation of `g`. Indeed, we can now see `g`
as a function that takes a function `factorial`, and returns an *implementation* for this function.
So what we need is another function `bla`, that takes something like `g`, and returns the inner
function type. The type of `bla` is, with appropriate bracketing:

    ((Int => Int) => (Int => Int)) => (Int => Int)

or, if we abstract over it:

    (T => T) => T

This is where the `fix` combinator comes in. The untyped definition of `fix`, for a call-by-value
evaluation semantics, is:

    fix = λf. (λx. f (λy. x x y)) (λx. f (λy. x x y))

A handful to write, a mouthful to read, and a mindful to understand! To get an intuition for how it
works, the best thing to do is to read/work through the example in TAPL, Chapter 5.2.

Mid summary
-----------

And therein lies my intuition about the existence of the fix combinator:

  * In lambda-calculus, we cannot recursively re-use a function name without
    abstracting over it
  * Once we abstract over it, we need a special function that *collapses* the
    abstracted type onto itself, or, indeed, calculates its fixed-point.

Newbie pitfalls
---------------

I used types in my reasoning above. Initially, I tried to go further, by (wrongly) attempting to type
`fix` in the simply-typed lambda-calculus. Fortunately Sandro enlightened me on the topic. To be
concise (see TAPL, ex. 9.3.2 for more details), there is no context `Γ`, type `T` such that

    Γ Ͱ x x : T

For a short proof sketch, in the above, `x` needs to be a function type and a value type at the same
time, which is impossible according to the context rules, where every value has at most one binding.
The fix combinator above faces this exact issue.

There are two solutions to this problem:

  * Either make `fix` part of the terms, by making it a primitive (TAPL 11.11) and adding
    evaluation and typing rules to mimic the behaviour of `fix` in a non-typed setting
  * Or, in the presence of more complicated type systems, for ex. with recursive types, we can
    implement fix using the base system (TAPL 20). You can find an attempt to implement it in Scala
    [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/ycombinator/Fixed.scala). It is an attempt, because in a
    call-by-value semantics, I have been forced to assume more on the type `T`. I'm happy to learn
    a better implementation!


The bottomline
--------------

Recursion is an interesting and weird concept. Either it is to be a primitive term, or can be recovered
from more complex/advanced type systems. Typically many languages choose the former. Languages with
advanced type systems naturally make it possible for the latter as well.

The code
--------

You can find the code for this post [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/ycombinator/Fixed.scala).

Related Readings
----------------

  * The TAPL book remains of course the reference for this.
  * There is also a wonderful [article](http://lampwww.epfl.ch/teaching/archive/foundations_of_programming/2001/tutorial/why_of_y.ps)
    that reconstructs the definition of the Y combinator from scratch.
