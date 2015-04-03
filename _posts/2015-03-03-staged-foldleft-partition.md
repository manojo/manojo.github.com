---
layout: post
title: "Staged FoldLeft and Partition"
description: ""
category:
tags: []
---
{% include JB/setup %}

Last [time]({% post_url 2015-02-19-staging-foldleft %}) we saw how to stage
`foldLeft` in LMS. We saw that this abstraction allows us to write pipelines of
operations over list-like functions, so that no intermediate lists are created.

In this post, I will look at the `partition` function, and study how we could
fuse it. I will try to take you through some partial, imperfect and downright
wrong ideas I have had, before eventually getting to what I think is a decent
solution, [with a little help from my friends](https://www.youtube.com/watch?v=jBDF04fQKtQ).

We will build on top of the staged `foldLeft` expression from last time. As usual,
you can look at the code (links at the end of this post).

The partition function
----------------------

The `partition` function on lists takes a list, a predicate, and returns two lists,
one containing elements satisfying the predicate, and the other those that do not.
We can implement this function using foldLeft:

{% highlight scala %}
def partition[A](ls: List[A], p: A => Boolean): (List[A], List[A]) =
  ls.foldLeft ((List[A](), List[A]())) { case ((trues, falses), elem) =>
    if (p(elem)) (trues ++ List(elem), falses)
    else (trues, falses ++ List(elem))
  }
{% endhighlight %}

The initial element is a pair of empty lists. At each iteration of the original
list, we either add the element to the first or the second list, based on the
predicate.

An example usage of partition is given here:

{% highlight scala %}
val myList: List[Int] = ...
val (evens, odds) = partition(myList, (x: Int) => x % 2 == 0)
(evens map (_ * 2), odds map (_ * 3))
{% endhighlight %}

In the context of fusion, we naturally want to avoid creating the `evens` and
`odds` lists.

A first attempt
---------------

Let's see how the above would work on the staged foldLeft abstraction. We can try
to first mimic the above type signature.

__Exercise 1__: Implement a function `partition` that creates a pair of `FoldLeft`s:

{% highlight scala %}
//inside the `FoldLeft` class
def partition(p: Rep[A] => Rep[Boolean]): (FoldLeft[A, S], FoldLeft[A, S]) = ???
{% endhighlight %}

The one way I can think of doing this is creating two foldlefts using the `filter`
function:

{% highlight scala %}
def partition(p: Rep[A] => Rep[Boolean]): (FoldLeft[A, S], FoldLeft[A, S]) = {
  val trues = this filter p
  val falses = this filter (a => !p(a))
  (trues, falses)
}
{% endhighlight %}

This looks good! Let us look at an example, to see what code is generated. Here's
the equivalent of the above example usage:

{% highlight scala %}
def partitionmapRange(a: Rep[Int], b: Rep[Int]): Rep[(List[Int], List[Int])] = {
  val xs = FoldLeft.fromRange[List[Int]](a, b)
  val (evens, odds) = xs.partition(_ % unit(2) == unit(0))
  val (mappedEvens, mappedOdds) = (evens map (_ * unit(2)) , odds map (_ * unit(3)))
  val evenList = (evens map (_ * unit(2))).apply(List[Int](), (ls, x) => ls ++ List(x))
  val oddList = (odds map (_ * unit(3))).apply(List[Int](), (ls, x) => ls ++ List(x))

  make_tuple2(evenList, oddList)
}
{% endhighlight %}

In addition to the building blocks we used last time, we are now using staged pairs
(aka `Rep[(A, B)]`). We have access to these through the `TupleOps` trait (please
look at the source). The `make_tuple2` function simply makes this tuple creation
explicit. Let us see what code this generates. It does not make much sense to post
the code here, it will break the reading flow. So please access it
[here](https://github.com/manojo/functadelic/blob/master/test-out/partition.check#L4).

We see that the `map` calls on partitioned lists get inlined as expected, so that's
good. Except that we have two iterations over the original list, instead of one. If
we take a closer look at the code, it is really not that surprising that we get two
loops. Indeed `FoldLeft` is an abstraction for evaluating an algebra, and for lists
(other collections) it is essentially an abstraction for a loop. Using the `filter`
function twice we get two naturally get two loops.

But generating two loops is not great: what if the range is really big, or if the
predicate function is computationally expensive? It really would be much better
to have a single loop. One possibility would be to have a lower-level transformation
that can combine two loops running on the same domain into a single loop. This is
known as vertical loop fusion \[[1][1]\]. But it feels a bit ironic to use another
fusion algorithm to implement our own fusion algorithm. We should be able to handle
this ourselves, or at worst argue why we cannot.

Yet another attempt
-------------------

Another possibility is to blindly follow the example list implementation, and
replace lists with `FoldLeft` as possible:

{% highlight scala %}
def partition(p: Rep[A] => Rep[Boolean]): Rep[(FoldLeft[A, S], FoldLeft[A, S])] =
  this.apply(/*what z*/){ /** what comb function */ }
{% endhighlight %}

Hmm. This also does not look like a great idea. What `z` value should we provide
to the `apply` function? We need something that is a staged pair of `FoldLeft`.
Which implies that we need to have some representation for `Rep[FoldLeft]`, and
know how to generate code for it. We could indeed do that, by creating specific
IR trees for `FoldLeft` and creating possible code generators for these trees[^1].

Once again though, this breaks the design of `FoldLeft` to some extent, because
we did intend it to be a unstaged encoding to start with. We should try to keep
the design to the extent that it is possible, and change only if we find out that
we cannot (once again arguing why it is not possible).

Partition bis: the use of ghost wrappers
----------------------------------------

The problem with partition is that while we fold the initial list, we want to push
the element to the correct downstream pipeline. As we saw in previous posts, `FoldLeft`
is push based, or a producer. The split point of the partition function acts as a
relay, which does not produce anything as much as relay elements to the right port.
One idea might be to flip things around so that a consumer abstraction takes over
at the receiving end:

![Consumer monster, image courtesy Nicolas]({{ site.url }}/images/consumer-monster.jpg){: .center-image}[^2]

As the above image portrays, that's yet another beast. Maybe we might need it for
other operations, but for `partition` we can do without it. Let us focus once
again on our objective: we want to generate a single loop. We already have an
abstraction for a loop, namely `FoldLeft`. So this fixes the return type for the
new partition function. Coming back to the partition function on lists, if we want
to produce a list (instead of a pair of lists), the type of the elements must be
something that captures the notion of left and right. Wait, but we have a type for
that, and it's called [`Either`](http://www.scala-lang.org/api/current/#scala.util.Either)!

__Exercise 2__: Implement `partition` on lists so that it return a `List[Either[A, A]]`:

{% highlight scala %}
def partition[A](ls: List[A], p: A => Boolean): List[Either[A, A]] = ???
{% endhighlight %}

The intro example then turns into the following:

{% highlight scala %}
val myList: List[Int] = ...
val partitioned: List[Either[Int, Int]] = partition(myList, (x: Int) => x % 2 == 0)
partitioned.map { x => x.map(_ * 2, _ * 3) }

//if we want to get two lists in the end
val (evens, odds) =  partitioned.foldLeft ((List[A](), List[A]())) {
  case ((trues, falses), elem) => elem match {
    case Left(x) => (trues ++ List(x), falses)
    case Right(x) => (trues, falses ++ List(x))
  }
}
{% endhighlight %}

For mapping on the partitioned elements, we use the `map` function on `Either`:
it takes two functions, one to apply to `Left` elements, one to `Right` elements.
If we do indeed need two lists at the end, we can fold the list into two separate
lists.

Let us step back for a second. What have we done? We have created an indirection
for the partitioning through a type that can differentiate two options. This way,
we still get to do the partitioning in one loop, but we end up creating extra boxed
data structures (of type `Either`). Which is also unnecessary overhead. You are
right... except that we are in a staging framework, so instead of using simple
`Either`, we can use its cousin staged `Either`. More generally, we can represent
`Either` as a struct with three fields (`left`, `right`, and a flag `isLeft`). And
then bank on underlying optimizations for structs to kick off, and eliminate them
completely.

Fortunately, staged structs and corresponding optimizations are implemented in
LMS \[[1][1]\], and you can take a look at them
[here](https://github.com/manojo/functadelic/blob/master/src/main/scala/lms/MyStructOps.scala).
By using all these underlying optimizations, it does indeed become possible to
implement partition for `FoldLeft` and get rid of the ghost wrappers at generation
time. Here is the implementation of the function:

{% highlight scala %}
def partitionBis(p: Rep[A] => Rep[Boolean]) =
  FoldLeft[Either[A, A], S] { (z: Rep[S], comb: Comb[Either[A, A], S]) =>
    this.apply(
      z,
      (acc, elem) =>
        if (p(elem)) comb(acc, left[A, A](elem))
        else comb(acc, right[A, A](elem))
    )
  }
{% endhighlight %}

In addition to the previous building blocks, we are now using the `EitherOps` trait
which handles `Either` as a struct in LMS. Please do take a look at the [test case](https://github.com/manojo/functadelic/blob/master/src/test/scala/barbedwire/FoldLeftSuite.scala#L179)
and corresponding [generated code](https://github.com/manojo/functadelic/blob/master/test-out/partition.check#L196)
to get a better understanding.

The bottomline
--------------

So there we go! It is possible to implement `partition` for `FoldLeft`, and have
no intermediate lists created. For this to work, we use ghost wrappers, and rely
on LMS optimizations to eliminate these wrappers later. This raises some questions:

__I thought we did not want to rely on anything and handle everything ourselves?__

Well, yes and no. We wanted to avoid having to create staged versions of our
`FoldLeft` abstraction, and we wanted to avoid using another fusion algorithm to
implement our fusion algorithm. We use optimizations on structs, which, with respect
to `FoldLeft`, is an arguably lower-level optimization. We can hence reasonably
argue that our initial goals are still achieved.

__Are we still in a short-cut setting?__

LMS enables two ways of rewriting/simplifying IR nodes:

  * the use of smart constructors and pattern matching on expressions to rewrite
  them. This is essentially what traits with the `ExpOpt` suffix do.
  * the use of the transformer framework.

In our case, we are only using the first way. Any rewrites which LMS performs are
therefore shortcut rewrites. They are shortcut rewrites on underlying structures,
not on `FoldLeft` or `List`. For example, some rewrites we perform are:

  * rewriting conditional expressions to one of the branches when the condition
  is a constant boolean.
  * rewriting field lookups on structs to variable access.

Please look at the code if you want an exhaustive list!

The code
--------

The code used in this post can be accessed through the following files:

  * The source implementation [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/barbedwire/FoldLeft.scala).
  * A test file [here](https://github.com/manojo/functadelic/blob/master/src/test/scala/barbedwire/FoldLeftSuite.scala).
  * A check file that contains the generated code [here](https://github.com/manojo/functadelic/blob/master/test-out/partition.check).
  * An implementation of `Either` as staged structs [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/lms/util/EitherOps.scala)
  * An implementation of staged structs [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/lms/MyStructOps.scala).
  Take a look at the `StructOpsExpOpt` and `StructOpsExpOptCommon` traits for the
  rewrite rules.
  * An implementation of staged conditional expressions
  [here](https://github.com/TiarkRompf/virtualization-lms-core/blob/develop/src/common/IfThenElse.scala).
  Take a look at the `IfThenElseExpOpt` trait for the rewrite rules.

If you want to run this code locally, please follow instructions for setup on
the readme [here](https://github.com/manojo/functadelic). The `sbt` command to
run the particular test is `test-only barbedwire.FoldLeftSuite`.


References
----------

1. [Optimizing Data Structures in High-Level Programs, Rompf et al.][1].
2. [Building-Blocks for Performance Oriented DSLs, Rompf et al., DSL 2011][2]

  [1]: http://dl.acm.org/citation.cfm?id=2429128
  [2]: http://arxiv.org/pdf/1109.0778

Footnotes
---------

[^1]: Check out the building blocks paper to understand the creation of IR nodes a bit better \[[2][2]\].
[^2]: Thanks [Nicolas](https://github.com/nicolasstucki) for the consumer monster!
