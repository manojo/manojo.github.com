---
layout: post
title: "Staged FoldLeft and GroupBy"
description: ""
category:
tags: []
---
{% include JB/setup %}

Last [time]({% post_url 2015-03-03-staged-foldleft-partition %}) we were able to
successfully add the `partition` function to the
[staged `FoldLeft` abstraction]({% post_url 2015-02-19-staging-foldleft %}). To
be precise, we were able to successfully write pipelines with partition, and have
them fused, so that no intermediate lists were created, and it was done in a single
traversal.

In this post we will consider a cousin of `partition`, the `groupBy` function.
While partitioning splits a list into two groups, `groupBy` partitions a list into
possibly many groups.


The `groupBy` function
--------------------

The `groupBy` function generalises `partition`. But that's not the only reason it
is interesting. It is a rather common query operation. It is of course used in query
languages, but it is also not uncommon in spreadsheet-like languages visualize
results better in spreadsheets. An example would be grouping a list of people by
their age groups, and then counting the number of people per group.

In Scala, `groupBy` takes a function as parameter; this function determines which
group an element belongs to. The results are aggregated in a `HashMap`.

__Exercise 1__: Implement the `groupBy` function for lists. Can you do it using
`foldLeft`?

{% highlight scala %}
def groupBy[A, K](ls: List[A], f: A => K): HashMap[K, List[A]] = ???
{% endhighlight %}

Using this function, we can implement a variant of the above example, where we
group a list of integers by their remainder in division by 3, and count the number
of elements in each group:

{% highlight scala %}
val groupedCount: HashMap[Int, Int] = groupBy(myList, a => a % 3) map {
  (k, v) => (k, v.sum)
}
{% endhighlight %}

Note that we use a map function on `HashMap` after the call to `groupBy`. Once
again, in terms of fusion, we would like to avoid creating the intermediate
`HashMap[Int, List[Int]]`.


Staging `groupBy`
-----------------

We saw last week that for `partition` we needed to introduce an extra box type
(`Either`) to represent values that satisfy the predicate (or not). The rationale
was to keep everything inside a single `FoldLeft`, because that represented exactly
one traversal.

This attempt to keep everything on one magical railway seems a good idea for `groupBy`
as well. Indeed, the construction of a list for elements of a group is completely
arbitrary. It makes sense to delay that construction as late as possible.

__Exercise 2__: What type should the elements of a group railway have?

The answer is that we should have pairs of type `(K, A)`. At the end of the day,
aka the final application of fold, we will decide whether to construct a `HashMap[Int, List[Int]]`
or a `HashMap[Int, Int]`. So what we have is a function on `FoldLeft` that we
will call `groupWith`:

{% highlight scala %}
//in FoldLeft

def groupWith[K: Manifest](f: Rep[A] => Rep[K]) =
  FoldLeft[(K, A), S] { (z: Rep[S], comb: Comb[(K, A), S]) =>
    this.apply(
      z,
      (acc, elem) => comb(acc, make_tuple2(f(elem), elem))
    )
  }
{% endhighlight %}


__Exercise 3__: Rewrite the `groupBy` followed by sum example using `groupWith`?

{% highlight scala %}
val groupedCount: HashMap[Int, Int] = ???
{% endhighlight %}

The above example now writes itself as:

{% highlight scala %}
val xs = FoldLeft.fromRange[HashMap[Int, Int]](a, b)
val grouped = xs.groupWith(x => x % unit(3))

val groupedCount = grouped.apply(
  HashMap[Int, Int](),
  (dict, x) =>
    if (dict.contains(x._1)) { dict + (x._1, dict(x._1) + x._2) }
    else { dict + (x._1, x._2) }
)
{% endhighlight %}

The bottomline: Paying dues to history
--------------------------------------

So that's it! We have been able to successfully add `groupBy` to our `FoldLeft`
abstraction as well! This is great, because now we can write query like pipelines,
and avoid allocating unnecessary structures as well. What's more, query languages
could use this `FoldLeft` abstraction to generate efficient queries as well! Of
course, they would need to do other optimizations such as re-ordering
filters/projections. But once that is done, here's a nice library to implement
the final step as!

Before we get ahead of ourselves, there are some questions it would be good to
answer first.

__I want to still write pipelines using `partition` and `groupBy` as if I were
using lists__

Fair point. I don't know how to answer this yet. I have a few ideas, which I can
hopefully develop in a future post!

__Have we extended [foldr/build fusion]({% post_url 2015-01-26-shortcut-fusion-part1 %})
in the process?__

It does indeed look as if we can deal with splitting operations, which, at first
sight, `foldr/build` fusion does not seem to handle. Hold your horses though. Check
out the following implementations of `partition` and `groupBy` for `FoldLeft`:

{% highlight scala %}
//inside FoldLeft

def partitionBis(p: Rep[A] => Rep[Boolean]): FoldLeft[Either[A, A], S] =
  this map { elem => if (p(elem)) left[A, A](elem) else right[A, A](elem) }

def groupWith[K: Manifest](f: Rep[A] => Rep[K]): FoldLeft[(K, A), S] =
  this map { elem => make_tuple2(f(elem), elem) }
{% endhighlight %}

That's right. Both of these functions are nothing but specialized applications of
`map` on `FoldLeft`. In my defense, this was not an intentional obfuscation, I
figured this out only now! As we know, `foldr/build` fusion handles `map` rather
well. So it should also handle `partition` and `groupWith/groupBy` OK.

Sad as that may sound, could we still consider ourselves clever? I would think so,
for the following reasons:

  * We have found a general way to handle splitting inside a `FoldLeft`: provide
  a function that maps an element to an appropriate type. It seems trivial in
  hindsight, but it did take us 2 full posts to get there.

  * We also know how to get rid of these "ghost" types (like `Either`) if we do
  not need them. We use the struct optimisations available in LMS. That explanation
  does not seem very satisfying however, so a better explanation is bound to be
  given soon!


The code
--------

The code used in this post can be accessed through the following files:

  * The source implementation [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/barbedwire/FoldLeft.scala).
  * A test file [here](https://github.com/manojo/functadelic/blob/master/src/test/scala/barbedwire/FoldLeftSuite.scala).
  * A check file that contains the generated code [here](https://github.com/manojo/functadelic/blob/master/test-out/partition.check).

If you want to run this code locally, please follow instructions for setup on
the readme [here](https://github.com/manojo/functadelic). The `sbt` command to
run the particular test is `test-only barbedwire.FoldLeftSuite`.


References
----------

No new references this time. Probably worth to take a look at the foldr/build fusion paper again.

1. 1. [A shortcut to deforestation, Gill et al., FPCA 1993][1].

  [1]: http://dl.acm.org/citation.cfm?id=165214
