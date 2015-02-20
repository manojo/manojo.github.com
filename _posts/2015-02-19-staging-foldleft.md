---
layout: post
title: "Staging FoldLeft"
description: ""
category:
tags: []
---
{% include JB/setup %}

In the series on shortcut fusion ([here]({% post_url 2015-01-26-shortcut-fusion-part1 %}) and [here]({% post_url 2015-02-11-shortcut-fusion-part2 %})) we
discovered two techniques to remove intermediate data structure in list
operation pipelines. This time, let's actually implement the very first of
these!

To be more precise:

  * We will consider `foldLeft` as an abstraction over list operations. This
  is because for lists, `foldLeft` and `foldRight` can be used to implement
  similar operations, and it turns out (as we will see) that `foldLeft` is a
  bit nicer to optimize in our scheme.
  * We will use staging, or [multi-stage programming](http://en.wikipedia.org/wiki/Multi-stage_programming)\[[1][1]\] as our base technique to
  evaluate intermediate structures away. In particular, we will use the LMS
  framework \[[2][2]\], \[[3][3]\].
  * As we saw in the previous posts, the shortcut rule is rather simple. The
  difficult part is optimizing the underlying structures after the shorcut
  rule has been applied. We will focus on this part, and not on the shorcut
  rule rewrites.


Staging and LMS
---------------

This post is not meant to be a detailed tutorial on how to use LMS. I will
however attempt to give just as much information as is required to understand
this post in a self-contained manner.

On that note, multi-stage programming (MSP) is a technique to separate the
compilation of a program into multiple stages. In an MSP setting, we explicitly
specify which parts of the program can be evaluated at a later stage (and by
corollary which parts evaluated now). Running such an annotated program `p0`
will yield a new program `p1`. This new program is behaviourally equivalent to
`p0`. The difference is that the parts that were not delayed have been
evaluated away, and the parts which were explicitly delayed are expressions in
`p1`. Essentially, MSP is a way to perform safe runtime code generation.

Closely related to MSP is the concept of [partial evaluation](http://en.wikipedia.org/wiki/Partial_evaluation). With this technique, if a program receives a
static and a dynamic input, the static part of the program is evaluated away, so
that the residual program (which still needs a dynamic input to yield a result)
is _specialized_ for the particular static input. This residual program, being
specialized, is expected to have better performance than to original program.
The main difference with MSP is that the static parts of the program are
_inferred_, rather than explicitly specified.

__LMS__

LMS is short for Lightweight Modular Staging. It is a staging/runtime code
generation framework in Scala. The execution of expressions is delayed through
the use of a special type constructor, `Rep[T]`. In other words:

  * An expression `val x: T = ...` is a constant in the generated code.
  * An expression `val x: Rep[T] = ...` is an expression of type `T` in the
  generated code.

The following diagram captures how to think of an LMS program:

![LMS]({{ site.url }}/images/lms.svg)

Instead of writing a program as in the left-bottom corner, we will be writing a
program containing `Rep` types, as in the top left corner. LMS will then take
care of going to the top-right corner, and beyond.

The type checker allows us to combine `Rep` expressions in a fairly seamless
way, so it feels like we are writing zero-stage programs. However, we are in
fact _composing code generators_ when we write an expression of type `Rep[T]`.
This is a very important concept to keep in mind for later on.

__Exercise 1__: What is the difference between `Rep[T => U]` and `Rep[T] => Rep[U]`?

To answer the question, let us desugar the types:

  * `Rep[T => U]` desugars to `Rep[Function[T, U]]`. This means that in the
  generated code (top-right corner) there will be a function.
  * `Rep[T] => Rep[U]` desugars to `Function[Rep[T], Rep[U]]`. This is an
  unstaged function. It only exists in the first stage. Applying it to an
  input of type `Rep[T]` effectively _inlines_ the code of the function at the
  application spot!

This is the one of the great tricks of writing LMS programs. We want to
generate code which contains little/no overhead. Generating functions when not
absolutely necessary is therefore futile. This thought is the driving force
behind designing a good staged library. Indeed, many higher order functions
can take unstaged functions as parameters, and we get inlining for free!

FoldLeft
--------

Armed with some knowledge about LMS, let us shift our focus to the main topic. The signature of `foldLeft` is given below:

{% highlight scala %}
def foldLeft[A, B](z: B, comb: (B, A) => A)(xs: List[A]) : B
{% endhighlight %}

It takes a zero element, a combination function, and applies it to a list. We have seen that the essence of `foldLeft` lies in the first parameter list: it can be applied to other collections as well. So we can abstract the second parameter list away for now. Using the guiding design principles, we come up with the following types:

{% highlight scala %}
trait FoldLefts extends ScalaOpsPkg with LiftVariables with While {

  /**
   * a type alias for the combination function for
   * foldLeft
   * `A` is the type of elements that pass through the fold
   * `S` is the type that is eventually computed
   */
  type Comb[A, S] = (Rep[S], Rep[A]) => Rep[S]

  /**
   * foldLeft is basically a pair of a zero value and a combination function
   */
  abstract class FoldLeft[A: Manifest, S: Manifest]
    extends ((Rep[S], Comb[A, S]) => Rep[S]) {
    ...
  }
}
{% endhighlight %}

The enclosing trait `FoldLefts` mixes in some of LMS' building blocks which help
in composing code generators\[[4][4]\]. In particular, we want to be able to
write a bit of mutable code (`LiftVariables`) and while loops (`While`). The
`Manifest` annotation on polymorphic types is specific to code generation.

__Exercise 2__: Pay close attention to the type of `abstract class FoldLeft`.
What is staged, and what is not?

As promised, we are using unstaged functions. Note that we are in fact also
using unstaged tuples. There is once again a subtle but all important difference
between `(Rep[A], Rep[S])` and `Rep[(A, S)]`.

The actual implementation of the `foldLeft` function for lists can be given as
follows:

{% highlight scala %}
/**
 * companion object, makes it easier to
 * construct folds
 */
object FoldLeft {

  /**
   * helper function for ease of use
   */
  def apply[A: Manifest, S: Manifest](f: (Rep[S], Comb[A, S]) => Rep[S]) =
    new FoldLeft[A, S] {

    def apply(z: Rep[S], comb: Comb[A, S]): Rep[S] = f(z, comb)
  }

  /**
   * create a fold from list
   */
  def fromList[A: Manifest, S: Manifest](ls: Rep[List[A]]) =
    FoldLeft[A, S] { (z: Rep[S], comb: Comb[A, S]) =>

    var tmpList = ls
    var tmp = z

    while (!tmpList.isEmpty) {
      tmp = comb(tmp, tmpList.head)
      tmpList = tmpList.tail
    }

    tmp
  }

  ...
}
{% endhighlight %}

I choose to implement it using while loops and mutable (but controlled)
variables rather than recursively. Why? I want to generate low-level code for
the JVM, which is better at optimizing while loops than recursive functions.
This also explains the choice of `foldLeft` over `foldRight`.

__Exercise 3__: Why does `fromList` take a `Rep[List[A]]` and not a `List[Rep[A]]`?

Well, because at the end of the day, we still need to provide an initial list
to the fold pipeline. And this list is an _actual_ input to the program, i.e.
it is unknown at staging time.

Let's make sure our code runs. What does that mean? It means:

  * High-level code written using `FoldLeft` generates lower-level code,
  containing no foldlefts.
  * The lower-level code runs on some input, and behaves as if the higher-level
  code ran on the same input.

Let's implement the identity function, for a sanity check:

{% highlight scala %}
/**
 * simple foldLeft back into a list
 */
def foldLeftId(in: Rep[Int]): Rep[List[Int]] = {
  val xs = List(unit(1), unit(2), unit(3))
  val fld = FoldLeft.fromList[Int, List[Int]](xs)

  fld.apply(List[Int](), (ls, x) => ls ++ List(x))

}
{% endhighlight %}

This function needs to be implemented in the right trait to get everything to
work properly. Please take a look at the full code (links in the code section
below). A few things to pay attention to:

  * The use of functions on lists here is in fact on `Rep[List]`, not on `List`.
  This is because we have inherited the ability to operate on `Rep[List]` from
  the `ScalaOpsPkg` trait.
  * You may laugh and scorn at the use of concatenation (`++`), which is
  basically algorithmic suicide. You are right. For really efficient code, we
  would use `ListBuffer` instead.
  * It seems a bit sad that we need to specify the type `S` as `List[Int]` so
  early in the pipeline, when we declare the `fld` value. Any ideas on how to
  improve upon that are welcome!

The generated code for `foldLeftId` has the following form:

{% highlight scala %}
val x1 = List(1,2,3)
var x3: scala.collection.immutable.List[Int] = x1
val x2 = List()
var x4: scala.collection.immutable.List[Int] = x2
val x18 = while ({val x5 = x3
  val x6 = x5.isEmpty
  val x7 = !x6
  x7
 }) {
  val x9 = x4
  val x10 = x3
  val x11 = x10.head
  val x12 = List(x11)
  val x13 = x9 ::: x12
  x4 = x13
  val x15 = x10.tail
  x3 = x15
  ()
}
val x19 = x4
x19
{% endhighlight %}

As we can see, we are left with a barebone while loop, exactly what we wished for.

__Exercise 4__: Write a function `fromRange` which creates an instance of
`foldLeft` over a range of integers:

{% highlight scala %}
def fromRange[S: Manifest](a: Rep[Int], b: Rep[Int]): FoldLeft[Int, S] = ???
{% endhighlight %}

Higher-order functions
----------------------

We are finally equipped to implement our good friends `map`, `filter`, `flatMap`.
Let me get you started with `map`:

{% highlight scala %}
def map[B: Manifest](f: Rep[A] => Rep[B]) =
  FoldLeft[B, S] { (z: Rep[S], comb: Comb[B, S]) =>
  this.apply(
    z,
    (acc: Rep[S], elem: Rep[A]) => comb(acc, f(elem))
  )
}
{% endhighlight %}

If you have read the previous post on foldr/build fusion, or implemented a `map`
on lists using fold, this should be obvious. The subtle point, however is that
the function taken as parameter is an unstaged function.
Calling `comb(acc, f(elem))` does not only inline the body of `comb`, but also
the body of `f`, in the _right_ place. This is where all the magic is actually
happening. Once again, please take a look at the appropriate test case (and
generated code) to make sure you get it.

__Exercise 5__: Write the `filter` function:

{% highlight scala %}
def filter(p: Rep[A] => Rep[Boolean]): FoldLeft[A, S] = ???
{% endhighlight %}

Once again, `p` is a staged function, so applying it inlines the body of the
predicate.

__Exercise 6__: Write the `flatMap` function:

{% highlight scala %}
def flatMap[B: Manifest](f: Rep[A] => FoldLeft[B, S]): FoldLeft[B, S] = ???
{% endhighlight %}

Hold on! What is the meaning of the type of `f`? Let's expand it:

{% highlight scala %}
f: Rep[A] => ((Rep[S], Comb[B, S]) => Rep[S])
{% endhighlight %}

We have a curried function. To be precise, we have a curried, unstaged function.
If we can fully apply the function (as opposed to partially applying it) we can
inline not only the body of `f`, but also the body of the resulting `FoldLeft`.
This way, we don't have to worry about generating code that represents a
`FoldLeft` after all! Here goes:

{% highlight scala %}
def flatMap[B: Manifest](f: Rep[A] => FoldLeft[B, S]) = FoldLeft[B, S] {
  (z: Rep[S], comb: Comb[B, S]) =>

  this.apply(
    z,
    (acc: Rep[S], elem: Rep[A]) => {
      val nestedFld = f(elem)
      nestedFld.apply(acc, comb)
    }
  )
}
{% endhighlight %}

As soon as we get a `nestedFld`, we immediately apply it, so all is well.

__Exercise 7__: What if the function passed to `flatMap` creates a `FoldLeft`
using `fromList`?

In that case, we do indeed have an issue: we will end up creating a list in the
generated code. So we should avoid writing such code. It's much better to use
something like `fromRange`, or even a `List[Rep[B]]`. Because the programmer is
expected to know the shape of the `FoldLeft` resulting from the `flatMap` call,
we can also reasonably expect her/him to use staged/unstaged structures
intelligently.

The bottomline
--------------

We have just implemented a compiler optimization for fusion! And we have done it
by writing almost only library-like code (modulo `Rep` types). I don't know
about you, but I find this rather elegant! We have done this by leveraging basic
MSP concepts, and the LMS framework. To be absolutely sure, you can take a look
at the test cases, where compositions of higher-order functions are implemented.
The generated code has the expected shape.

In terms of fusion algorithms, how powerful are we? Well, we are no more
powerful than foldr/build fusion: we face the same issues with `zip` as them. We
are also at least as powerful as them: we can fuse all the functions they can.
To convince yourself, you could implement `concat` as an exercise.

So it seems we are as powerful as them (we might need to prove this more formally
). We are arguably more elegant, because we do not rely on beta-reduction or
other underlying compiler optimizations. We have implemented a staged library
instead!

The code
--------

The code used in this post can be accessed through the following files:

  * The source implementation [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/barbedwire/FoldLeft.scala).
  * A test file [here](https://github.com/manojo/functadelic/blob/master/src/test/scala/barbedwire/FoldLeftSuite.scala).
  * A check file that contains the generated code [here](https://github.com/manojo/functadelic/blob/master/test-out/foldleft.check).

If you want to run this code locally, please follow instructions for setup on
the readme [here](https://github.com/manojo/functadelic). The `sbt` command to
run the particular test is `test-only barbedwire.FoldLeftSuite`.


References
----------

1. [Multistage programming][1], the official home page.
2. [Lightweight Modular Staging][2], the official entry page.
3. [Lightweight modular staging: a pragmatic approach to runtime code generation and compiled DSLs, Rompf et al., CACM 2012][3]
4. [Building-Blocks for Performance Oriented DSLs, Rompf et al., DSL 2011][4]

  [1]: http://www.cs.rice.edu/~taha/MSP/
  [2]: http://scala-lms.github.io
  [3]: http://dl.acm.org/citation.cfm?id=2184345
  [4]: http://arxiv.org/pdf/1109.0778
