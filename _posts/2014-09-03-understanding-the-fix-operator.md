---
layout: post
title: "Understanding the fix operator"
description: ""
category:
tags: []
---
{% include JB/setup %}

I finally got some intuition for the [fixed-point combinator](http://en.wikipedia.org/wiki/Fixed-point_combinator).
To be more precise, I got an intuition for why it exists, not why it's defined as it is. Here's how I understood it:

## Recursion in Lambda-Calculus

Let us try to write a simple recursive function, factorial, in the Lambda-Calculus with some extensions
for integers and conditionals. Because I know how to write it in Scala, here is a Scala implementation:

{% highlight scala %}
def factorial(n: Int): Int = if (n == 0) 1 else n * factorial(n - 1)
{% endhighlight %}

Here's a first attempt in lambda-calculus:

    factorial = λn. if iszero n then 0 else factorial ...

But wait! We cannot re-use `factorial` because it is not a function name in the definition! The only
way to re-use it is to abstract over it:

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
works, the best thing to do is to read/work through the example in TAPL, Chapter 5.2. I'm going to
trust that the definition is correct. Here let's just assure ourselves that we do indeed satisfy the
type signature above. Running the type inference algorithm in our head, we have no choice but to
assign `T => T` for `f`. Let's look at

    bar = λx. f (λy. x x y)

We know that `f (λy. x x y)` needs to be of type `T`, because it is an application of `f`. Therefore,
`x` has to be of type `T => T`, and `y` of type `T`. This means that `bar` has type `T => T`. It is
applied to itself:

    (T => T)(T => T)

The return type is naturally `T => T`, and so everything works out.

## The bottomline

We need `fix` because:

  * In lambda-calculus, we cannot recursively re-use a function name without abstracting over it
  * Once we abstract over it, we need a special function that *collapses* the abstracted type onto
  itself, or, indeed, calculates its fixed-point.

Armed with this new intuition, a good way to get hands dirty is to solve the exercises in TAPL,
Chapter 5. How does mutual recursion work?

