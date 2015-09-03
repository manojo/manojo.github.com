---
layout: post
title: "Staged Parser Combinators and Recursion"
description: ""
category:
tags: []
---

In the [last post]({% post_url 2015-09-02-staged-parser-combinators %}) we saw
how to stage parser combinators. We added alternation and concatenation, and
CPS-encoded the parse result datas tructure to avoid intermediate allocations for
it.

One important thing we didn't do yet is recursion. Indeed, many data formats
(JSON, programming languages, serialization formats) have a nested, self-similar
structure. And as we are staging parser combinators, we should be able to stage
recursive parsers too. Which is what we will do in this post. For once, we will
slightly steer away from our principle of doing everything as a library, and use
some minimal LMS internals.

A Playful Diversion
-------------------

But first, a cute little exercise, thanks to [Alex](https://axel22.github.io/),
in which we explore recursion a bit. We all know how to write the Fibonacci function:

{% highlight scala %}
def fib(n: Int): Int = if (n == 0 || n == 1) 1 else fib(n - 1) + fib(n - 2)
{% endhighlight %}

The problem with this implementation is that it computes many values more than
once. For example, here is the trace for `fib(4)`

    fib(4)
    ↪ fib(3) + fib(2)
    ↪ fib(2) + fib(1) + fib(2)
    ↪ fib(1) + fib(0) + fib(1) + fib(2)
    ↪ ...

We compute `fib(2)` twice, which is not necessary. We could for instance memo-ize
results as we go instead.

__Exercise 1__: Suppose you are given a cache in scope. Reimplement `fib` so that
it memo-izes intermediate results.

{% highlight scala %}
val cache = scala.collection.mutable.HashMap.empty[Int, Int]
def memofib(n: Int) = ???
{% endhighlight %}

Now we may have more than one function that we would like to memo-ize. It is
tedious to have to create a cache for each such function. Could we do better?

__Exercise 2__: Define a function `memo` that takes a function (`Int => Int`) and
returns a memo-ized version of it. You can use caches if need be. Reimplement `fib`
using `memo`.

{% highlight scala %}
def memo(f: Int => Int): Int => Int = ???
{% endhighlight %}

It's probably easier to implement backwards. The real trick lies in implementing
`fib` using `memo`. Following the types, we can get to something like:

{% highlight scala %}
def fib: Int => Int = memo { i => if (i == 0 || i == 1) 1 else ??? }
{% endhighlight %}

And then we realize that we need to refer to `fib` in the expression we pass to
`memo`: after all, we are trying to implement a recursive function. But, and here
lies the trick, we need to refer to the _same_ `fib` that we declare. And for that
we cannot declare `fib` using a `def`. It has to be the _same_ function value,
and `def` creates a new value at every invocation. And we therefore have few options
other than resorting to laziness:

{% highlight scala %}
lazy val fib: Int => Int = memo { i => if (i == 0 || i == 1) 1 else fib(i - 1) + fib(i - 2) }
{% endhighlight %}

Indeed, thanks to laziness, the first time we use `fib` a reference for it will
be created and then properly bound to further uses of `fib`. As a result any
internal structures that `memo` uses will be bound to that reference. And that
should definitely give you enough info to implement `memo` now!


Recursion and Staging
---------------------

So what is the problem with recursion and staging? Seems that it should work out
of the box? Well, consider the following recursive parser, which parses digits
and adds them up:




Indeed, parsers are extremely
handy for nested, self-similar formats. In fact, if the data we want to parse
is not recursive in structure, we can do the job with state


We previously [observed]({% post_url 2015-02-19-staging-foldleft %}), a long time
ago, that combining partial evaluation and CPS-encoded data structures enabled us
to achieve fusion. To be a bit elaborate:

  1. If we CPS-encode data structures, fusion on these structures reduces to fusion
  on function composition.
  2. By partially evaluating function composition we effectively achieve fusion.

In this post, we will explore another interesting application of this principle,
parser combinators \[[1][1]\]. Essentially, through the use of staging, we will
turn parser combinators into parser generators. The latter are rid of the composition
overhead of combinators, and therefore have good runtime performance \[[2][2]\].

This post heavily borrows from the lessons learnt on the previous adventure with
fold-based fusion. I will therefore assume that the reader is familiar with the
staging techniques we have used so far. I'd recommend reading the paragraph on
LMS over [here]({% post_url 2015-02-19-staging-foldleft %}) in order to get an idea.
