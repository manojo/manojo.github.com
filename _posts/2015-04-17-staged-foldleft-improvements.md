---
layout: post
title: "Code improvements to Staged FoldLeft"
description: ""
category:
tags: []
---
{% include JB/setup %}

Over the last few posts, we explored [`foldr/build` fusion]({% post_url 2015-01-26-shortcut-fusion-part1 %})
in the context of staging and partial evaluation. We figured out that:

  1. If we CPS-encode data structures, fusion on these structures reduces to fusion
  on function composition.
  2. By partially evaluating function composition we effectively achieve fusion.

Using these insights, we encoded lists as `FoldLeft`, its CPS-encoding. By staging
`FoldLeft` we were able to achieve [fusion]({% post_url 2015-02-19-staging-foldleft %}).
We then added multiple producer functions to the abstraction, namely [`partition`]({% post_url 2015-03-03-staged-foldleft-partition %})
and [`groupBy`]({% post_url 2015-03-12-staged-foldleft-groupby %}). To keep everything
on a single pipeline we had to add extra boxes. But then, these boxes could also
be CPS-encoded, and we were able to get rid of them as [well]({% post_url 2015-03-20-cps-encoding-either %}).

Over the last few weeks, I packed all of this up into a two column [PDF]({{ site.url }}/resources/staged-fold-fusion.pdf). Note: it's
under submission. During the process, we (along with Sandro) figured out better
(and even safer) ways to implement the library. In this shortish post, I mention
these improvements. If you read through the PDF you might not notice every one of
them, but if you have been following the past posts (and have been doing some of
the exercises), it is hopefully useful.

A Better FoldLeft
-----------------

Initially, we gave the following signature to `FoldLeft`:

{% highlight scala %}
abstract class FoldLeft[A: Manifest, S: Manifest]
  extends ((Rep[S], Comb[A, S]) => Rep[S]) {
  ...
}
{% endhighlight %}

With this signature, we had to signal very early in the pipeline what the eventual
data structure we folded into would be. In many of our examples, we turned `S`
to `List[Int]`. Because the fold is applied only at the end, it would make more
sense to specify `S` only then. With the following type signature for `FoldLeft`
we get exactly that:

{% highlight scala %}
abstract class FoldLeft[A: Manifest] { self =>
  def apply[S: Manifest](z: Rep[S], comb: Comb[A, S]): Rep[S]
  ...
}
{% endhighlight %}

Now `FoldLeft` is parametrized over `A`, the type of elements it folds over. Its
`apply` function is paremtrized over the eventual fold type `S`. Thanks to this
modification we can write fold pipelines without having to worry about the final
type until the end.


Code generation for Conditional Expressions
-------------------------------------------
Recall that we had to define a specific conditional combinator over `EitherCPS`:

{% highlight scala %}
def conditional[A: Manifest, B: Manifest](
  cond: Rep[Boolean],
  thenp: => EitherCPS[A, B],
  elsep: => EitherCPS[A, B]
): EitherCPS[A, B] = { ... }
{% endhighlight %}

This was because `EitherCPS` is a code generator, and we needed to push the evaluation
of the conditional expression to code generation. Our previous implementation was
naive however:

{% highlight scala %}
def conditional[A: Manifest, B: Manifest](
  cond: Rep[Boolean],
  thenp: => EitherCPS[A, B],
  elsep: => EitherCPS[A, B]
): EitherCPS[A, B] = new EitherCPS[A, B] {
  def apply[X: Manifest](lf: Rep[A] => Rep[X], rf: Rep[B] => Rep[X]) =
    if (cond) thenp.apply(lf, rf) else elsep.apply(lf, rf)
}
{% endhighlight %}

The problem with this implementation is that it does not scale with respect to
code generation, especially if nested `EitherCPS` instances are used.

__Exercise__: What does the following code generate :

{% highlight scala %}
val c1: Rep[Boolean] = ...
val c2: Rep[Boolean] = ...

val c =
  if (cond1)
    if (cond2) mkLeft[Int, String](1)
    else mkRight[Int, String]("hi")
  else
    if (cond2) mkLeft[Int, String](3)
    else mkRight[Int, String]("hello")
c.apply(lf, rf)
{% endhighlight %}

We get the following evaluation trail:

    (if (c1) { if (c2) t else e } else { if (c2) t2 else e2 }).apply(lf, rf)
    ↪ if (c1) { if (c2) t else e }.apply(lf, rf) else { if (c2) t2 else e2 }.apply(lf, rf)
    ↪ if (c1) {
        if (c2) t.apply(lf, rf) else e.apply(lf, rf)
      } else {
        if (c2) t2.apply(lf, rf) else e2.apply(lf, rf)
      }

The `apply` function is inlined 4 times!! This is not a problem specific to `EitherCPS`,
but applies to conditional expressions and code generation in general. LMS provides
a generic way to solve the problem, with the use of so called
[`FatIfThenElseExp`](https://github.com/TiarkRompf/virtualization-lms-core/blob/develop/src/common/IfThenElse.scala#L129).
In our case, we can also provide an alternate implementation for `conditional`:

{% highlight scala %}
def conditional[A: Manifest, B: Manifest](
  cond: Rep[Boolean],
  thenp: => EitherCPS[A, B],
  elsep: => EitherCPS[A, B]
): EitherCPS[A, B] = {
  import lms.ZeroVal

  var l = ZeroVal[A]; var r = ZeroVal[B]
  var isLeft = true
  val lCont = (a: Rep[A]) => { l = a; isLeft = true }
  val rCont = (b: Rep[B]) => { r = b; isLeft = false }

  if (cond) thenp.apply[Unit](lCont, rCont)
  else elsep.apply[Unit](lCont, rCont)

  new EitherCPS[A, B] {
    def apply[X: Manifest](lf: Rep[A] => Rep[X], rf: Rep[B] => Rep[X])
                          (implicit pos: SourceContext) = {
      if (isLeft) lf(l) else rf(r)
    }
  }
}
{% endhighlight %}

The idea is to call a continuation which stores temporarily stores results. In the
conditional expression we just pass these temporarily stored values.

The bottomline
--------------

That's it! In this fairly short post we improved the library look and feel, and
the code generation for the staged `FoldLeft` API. Code generation for conditional
expressions can get tricky, it is important to keep the above problem in mind!


The code
--------

The code now lives in its own little repo [here](https://github.com/manojo/staged-fold-fusion/).
Follow the instructions on the README if you want to run the code!


References
----------

No references today, sorry!

