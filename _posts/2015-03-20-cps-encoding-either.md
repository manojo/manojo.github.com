---
layout: post
title: "Of Either and CPS encodings thereof"
description: ""
category:
tags: []
---
{% include JB/setup %}

In the last few posts, we were able to integrate the [`partition`]({% post_url 2015-03-03-staged-foldleft-partition %})
and [`groupBy`]({% post_url 2015-03-12-staged-foldleft-groupby %}) functions into
the [staged `FoldLeft` abstraction]({% post_url 2015-02-19-staging-foldleft %}),
and were able to perform short-cut fusion on them. The trick was to find versions
of these functions which could _preserve_ the `FoldLeft` representation, so that
the folding need be only eventually done. These functions yielded instances of
`FoldLeft` on slightly more complex types (`Either[A, A]` for `partition`, `Pair[K, A]`
for `groupBy`).

We conveniently swept the issue of removing instances of these ghost types under
the carpet. The justification being that "LMS takes care of it, trust me, it works".
In this post, we will take a look under that carpet. To be precise, we will go back
to some first principles, and see how staging these will naturally give us the
intermediate structure removal "optimization" as a library.

Of CPS Encodings
----------------

Let us look at the modified implementation of `partition` once again:

{% highlight scala %}
//inside FoldLeft

def partitionBis(p: Rep[A] => Rep[Boolean]): FoldLeft[Either[A, A], S] =
  this map { elem => if (p(elem)) left[A, A](elem) else right[A, A](elem) }
{% endhighlight %}

Thanks to this implementation, we can continue piping operations until we are
satisfied. The operations are over `Either` types though. Last time, we used a
[struct representation](https://github.com/manojo/functadelic/blob/master/src/main/scala/lms/util/EitherOps.scala)
for `Either`, in order to generate simple structs.

Whether we decided to generate simple structs or case classes, the point is that
we still committed too early to a representation for `Either`. Because we were
applying the instance of `FoldLeft` to accumulate into a pair of lists, we really
didn't need to have any instances of `Either`. The question therefore is whether
we can somehow delay the representation for our type a bit further.

__Exercise 1__: Can you think of a way to encode the delayed representation for `Either`?

If you can, then congratulations, you don't need to read the post any further! If
you can't yet, don't fret. Think back to one of the basic tricks of functional
programming we learnt: when in doubt, try creating an extra function abstraction,
and later applying it.

If we do that for `Either`, we get the following representation:

{% highlight scala %}
abstract class EitherCPS[A, B] { self =>
  def apply[X](lf: A => X, rf: B => X): X
}
{% endhighlight %}

Here's a more mathematical notation:

    type EitherCPS[A, B] = forall X. (A => X, B => X) => X

Essentially, `EitherCPS` _is_ the function that abstracts over the eventual representation,
or `X`. It takes two functions that represent the left and the right side. Naturally
they are also function types. When applied, they will yield the eventual representation
`X`.

This representation is also known as a CPS encoding \[[1][1]\]: the representation
`X` is concretized when a continuation is passed to an instance of `EitherCPS`.
How do we create `Left` and `Right` instances? As with the classic `Either`, we
can have case classes representing the left and the right variants.

__Exercise 2__: Implement the `apply` function for `LeftCPS` and `RightCPS`:

{% highlight scala %}
case class LeftCPS[A, B](a: A) extends EitherCPS[A, B] {
  def apply[X](left: A => X, right: B => X) = ???
}

case class RightCPS[A, B](b: B) extends EitherCPS[A, B] {
  def apply[X](left: A => X, right: B => X) = ???
}
{% endhighlight %}

The `apply` functions above act as element carriers. When, eventually, the instance
of `EitherCPS` is applied, the element that is carried is used, and applied to the
left (or right) function.

For convenience sake (and for some more practice), let us also implement the `map`
function:

__Exercise 3__: Implement a `map` function on `EitherCPS`, analogous to the `map`
on `Either`:

{% highlight scala %}
def map[C, D](lmap: A => C, rmap: B => D): EitherCPS[C, D] = ???
{% endhighlight %}

This may look tricky at first, but if we follow the types, it is actually quite
easy:

{% highlight scala %}
def map[C, D](lmap: A => C, rmap: B => D) = new EitherCPS[C, D] {
  def apply[X](lf: C => X, rf: D => X) = self.apply(
    a => lf(lmap(a)),
    b => rf(rmap(b))
  )
}
{% endhighlight %}

As a final practice exercise, let us implement our favorite `partition` function
on lists:

__Exercise 3__: Implement the `partition` function on lists. It should first create
an intermediate list of `EitherCPS`, and then fold the result of that list into
a pair of lists:

{% highlight scala %}
def partition[A](ls: List[A], p: A => Boolean): (List[A], List[A]) = {
  val tmp: List[EitherCPS[A, A]] = ???
  tmp.foldLeft(???) { ??? }
}{% endhighlight %}

The only tricky part is passing the correct continuation to every `EitherCPS`
element in the temporary list:

{% highlight scala %}
def partition[A](ls: List[A], p: A => Boolean): (List[A], List[A]) = {
  val tmp: List[EitherCPS[A, A]] = ls map { a =>
    if (p(a)) LeftCPS[A, A](a) else RightCPS[A, A](a)
  }

  tmp.foldLeft ((List[A](), List[A]()) { case ((trues, falses), elem) =>
    elem.apply[(List[A], List[A])](
      l => (trues ++ List(l), falses),
      r => (trues, falses ++ List(r))
    )
  }
}
{% endhighlight %}

We pass two functions, one which adds an element to the left accumulator, one to
the right.

Checkpoint
----------

Before we go further, let us shortly reflect on `EitherCPS`. This library looks
more complicated than the classic one for `Either`: what exactly have we achieved?
We have created a representation for `Either` that delays its actual construction.
In the above example, note that when we fold into the final list, we do not pattern
match on an actual instance of `Either`: rather we call the `apply` method, which
will "do the right thing". But, arguably, we are still creating instances of `EitherCPS`
after all, aren't we? Before solving that problem, let us quickly deepen our intuition
about CPS encodings.

__Exercise 4__: Can you come up with a mathematical notation (see above) which CPS
encodes lists? Hint: you may have seen it [here]({% post_url 2015-01-15-essence-of-fold %})
before.

    type List[A] = forall X. ???

It turns out to be nothing but the list functor:

    type List[A] = forall X. (() => X, (A, X) => X) => X

Does this remind you of something else? You are right, this is the type signature
of `foldRight`! And so we have come full circle here:

  * We started out by representing list operations as fold operations. We thought
  this was a good idea because of the essence of fold, i.e. the presence of a unique
  morphism from the `Fix` algebra to any other algebra. We took the list functor
  for granted at that time, as we were only interested in operating with fold.

  * We got stuck at some point because some operations didn't fit on the `fold`
  railway. They created unnecessary bloat.

  * We got unstuck by remembering to go back to the basics (or functor representations),
  and use their very basic version, also known as CPS encodings.

Staging EitherCPS
-----------------

If you have survived the discussion so far, you may have guessed where we are going
with this. Just like `FoldLeft`, `EitherCPS` is also a function (or abstract) representation
of something else. Which means we can use the same staging technique to get rid
of any instance of `EitherCPS` as well.

Because the staging itself is straightforward (especially as we have done it
[before]({% post_url 2015-02-19-staging-foldleft %})), we will not discuss it here.
Please take a look at the code for details. Maybe just do this exercise anyway:

__Exercise 5__: Add appropriate `Rep` types to `EitherCPS` above to stage it.


Tying the knot
--------------

Getting back to `FoldLeft`, we face one final issue. Here's the implementation of
`partition` using `EitherCPS`:

{% highlight scala %}
def partitionCPS(p: Rep[A] => Rep[Boolean]): FoldLeft[EitherCPS[A, A], S] =
  this map { elem =>
    if (p(elem)) LeftCPS[A, A](elem) else RightCPS[A, A](elem)
  }
{% endhighlight %}

The compiler will tell us that it is expecting a `Rep[EitherCPS[A, A]]` when we
give it an `EitherCPS`. We have no choice unfortunately than to create an IR node
that represents this type. Luckily, _because_ we know that we will never need to
generate code for a `Rep[EitherCPS[A, B]]`, all we need is to do is add wrappers
around `EitherCPS`:

{% highlight scala %}
trait EitherCPSOpsExp extends EitherCPSOps
    with BaseExp {

  case class EitherWrapper[A, B](e: EitherCPS[A, B]) extends Def[EitherBis[A, B]]
}
{% endhighlight %}

The `EitherCPSOpsExp` extends the `BaseExp` trait, which in LMS represents the world
of expressions. In this world, values of type `Rep[T]` are converted into IR nodes
(of type `Exp[T]` or `Def[T]`) that represent them. In the above, we have created
a class that extends `Def[EitherBis[A, B]]`: we create a type `EitherBis` to distinguish
the "repped" type from the CPS representation.

In addition to this, we need to create wrappers for `apply`, `map`, so that these
operations are also admitted on `Rep[EitherBis[A, B]]`. This is straightforward
building blocks stuff in LMS \[[2][2]\], so please take a look at the code. I will
just provide the final implementation of partition here:

{% highlight scala %}
def partitionCPS(p: Rep[A] => Rep[Boolean]): FoldLeft[EitherBis[A, A], S] =
  this map { elem =>
    either_conditional(p(elem), mkLeft[A, A](elem), mkRight[A, A](elem))
  }
{% endhighlight %}

Conditional Expressions
-----------------------

Before wrapping up, you may have noticed above that there is one final twist. Consider
the following expression:

{% highlight scala %}
val c = if (p(elem)) mkLeft[A, A](elem) else mkRight[A, A](elem)
c.apply(lf, rf)
{% endhighlight %}

LMS uses (originally used) Scala-virtualized to virtualize Scala expressions, i.e.
convert them to method calls \[[3][3]\]. For the above example, the conditional
expression is converted to a call to the `__ifThenElse` method, which has the
following signature:

{% highlight scala %}
def __ifThenElse[T: Manifest](
  cond: Rep[Boolean],
  thenp: => Rep[T],
  elsep => Rep[T]
): Rep[T]
{% endhighlight %}

While in an unstaged setting we would have the above example evaluate to :

{% highlight scala %}
if (p(elem)) mkLeft[A, A](elem).apply(lf, rf)
else mkRight[A, A](elem).apply(lf, rf)
{% endhighlight %}

In the staged setting, an IR node for a conditional expression is created. Which
means we must introduce an explicit rule for evaluating conditional expressions
that yield a `Rep[EitherBis[A, B]]`. Hence the call to `either_conditional` in the
`partitionCPS` code. The implementation is given below:

{% highlight scala %}
def either_conditional[A: Manifest, B: Manifest](
  cond: Rep[Boolean],
  thenp: => Rep[EitherBis[A, B]],
  elsep: => Rep[EitherBis[A, B]]
): Rep[EitherBis[A, B]] = (thenp, elsep) match { //stricting them here
  case (Def(EitherWrapper(t)), Def(EitherWrapper(e))) =>
    EitherWrapper(conditional(cond, t, e))
}

//in EitherCPS..
def conditional[A: Manifest, B: Manifest](
  cond: Rep[Boolean],
  thenp: => EitherCPS[A, B],
  elsep: => EitherCPS[A, B]
): EitherCPS[A, B] = new EitherCPS[A, B] {
  def apply[X: Manifest](lf: Rep[A] => Rep[X], rf: Rep[B] => Rep[X]) =
    if (cond) thenp.apply(lf, rf) else elsep.apply(lf, rf)
}
{% endhighlight %}

This way, we tie the final knot! Maybe we could have a general representation for
mixed stage conditional expressions, so that we don't need to re-implement it every
time.

The bottomline
--------------

So that is it! We have successfully dug under the carpet, and come up with an encoding
for `Either` that allows us to optimize it. The nice thing that comes out of this
is that using staging and CPS encodings, we can bring a lot of optimizations to
the library level. One may argue that compilers can do all of this already. The
counter-argument I can think of is that because we have it available at the library
level, it is easier for a DSL developer to control and choose which optimizations
he wants!


The code
--------

The code used in this post can be accessed through the following files:

  * The implementation of `CPSEither` [here](https://github.com/manojo/staged-fold-fusion/blob/master/src/main/scala/barbedwire/FoldLeft.scala). There is a vanilla Scala (aka non-LMS) version [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/cpsencodings/EitherCPS.scala).
  * The `partition` function using the CPS encoding [here](https://github.com/manojo/staged-fold-fusion/blob/master/src/main/scala/lms/util/EitherCPSOps.scala).
  * A test file [here](https://github.com/manojo/staged-fold-fusion/blob/master/src/test/scala/barbedwire/FoldLeftSuite.scala).
  * A check file that contains the generated code [here](https://github.com/manojo/staged-fold-fusion/blob/master/test-out/partition.check).

If you want to run this code locally, please follow instructions for setup on
the readme [here](https://github.com/manojo/staged-fold-fusion). The `sbt` command to
run the particular test is `test-only barbedwire.FoldLeftSuite`.


References
----------

1. [Continuation Passing style][1]
2. [Building-Blocks for Performance Oriented DSLs, Rompf et al., DSL 2011][2]
3. [Scala-Virtualized: Linguistic Reuse for Deep Embeddings, Rompf et al., CACM 2013][3]

  [1]: en.wikipedia.org/wiki/Continuation-passing_style
  [2]: http://arxiv.org/pdf/1109.0778
  [3]: http://lampwww.epfl.ch/~amin/pub/hosc2013.pdf


