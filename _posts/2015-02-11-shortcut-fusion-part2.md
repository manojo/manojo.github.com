---
layout: post
title: "Shortcut Fusion - Part 2"
description: ""
category:
tags: []
---
{% include JB/setup %}

Last [time]({% post_url 2015-01-26-shortcut-fusion-part1 %}), we discovered `foldr/build` fusion, a
technique to remove intermediate lists in a pipeline of list operations. I initially wanted to talk
about its dual, `unfoldr/destroy` fusion \[[1][1]\], before coming to this, but I think we can skip
straight ahead to Stream Fusion \[[3][3]\].

Fusion redux
------------

Just to recap, the idea of fusion is to remove intermediate datastructures created in a pipeline of
operations. We would also like our fusion algorithm to have the following desirable properties:

  1. The algorithm should work for as many different list operations as possible.
  2. The algorithm should be simple, elegant even.


Theoretical intuition
---------------------

We saw last time that `foldr/build` fusion clearly ticked the second box above. However, functions
like `zip` didn't fuse well with this algorithm. This is because `zip` is not well expressed as a
`foldRight` operation: `foldRight` is an inherently push-like operation, whereas zipping two lists
requires us to pull elements from 2 lists before packaging and pushing a pair further downstream.

**Exercise 1**: Why is `foldRight` push-based?

Let's look at the signature of it again:

{% highlight scala %}
def foldRight[A, B](z: B)(comb: (A, B) => B)(ls: List[A]): B
{% endhighlight %}

The answer lies in the signature of `comb`. It receives and element of type `A`, and has to do
something with this element. It "pushes" this element further by applying the `comb` function.

So what if we used a pull-based abstraction over lists instead? That is indeed the idea behind
`destroy/unfoldr` fusion (which we won't cover). In that case, zips become easy again, but in this
abstraction, fusion becomes tricky for `filter`. In a pull-based abstraction, `filter` is
implemented in a recursive manner. To get some intuition why, I'd recommend reading up the
implemented of this function for Scala's [`Iterator`](https://github.com/scala/scala/blob/v2.11.5/src/library/scala/collection/Iterator.scala#L407).

The bottomline is that whatever the abstraction, it is easy to perform fusion if list operations are
implemented as processing one element at a time. A recursive implementation of `filter` implies that
we may process many elements in one single go, and this makes it difficult to fuse.

Enter Stream Fusion
-------------------

So the problem is that in both cases, some functions don't fit the abstraction quite well. Enter
Stream Fusion, or streams, to be more precise. Note: these are not streams in the [Scala sense](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Stream) of it.

The stream abstraction lies somewhat in the middle of the push/pull divide. One can think of it as a
magical railway: elements pass through a stream, or a railway track. Once in a while, these elements
may slip and fall, at which point the track may choose to create ghost elements to push further down
the track.

Before looking at the implementation of streams, we can try to see how stream fusion works. There
are two steps to it:

**Part 1: Lists to Streams**

Convert a list to a stream, and run the equivalent list operation on the stream instead, before
finally converting back to a list. For example,

    map(ls, f)

is rewritten as

    mkList(map_s(mkStream(xs), f))

where the `_s` suffix stands for the operation on streams. Then, apply the following shortcut rule:

    mkStream(mkList(bla)) ---> bla


**Exercise 2**: Verify that the above rule gets rid of intermediate lists in a sequence of map
functions:

    map((map(xs, f)), g)


**Part 2: Streams to "Nothing at all"**

Once there are only operations on streams left, the underlying compiler (in this case, Haskell) can
eliminate streams, and any other intermediate objects, altogether. We will not be seeing how this is
done in this post, unfortunately. I hope to be able to show you that sometime soon in the future!

In the rest of this article, we will acquaint ourselves with the stream data structure, at least to
get an intuition of why it's so easy to eliminate later on.

Streams themselves
------------------

The signature for the Stream datastructure is given below:

{% highlight scala %}
trait Streams {

  /** ADT for a Stepper */
  abstract class Step[A,S]
  case class Done[A,S]() extends Step[A,S]
  case class Yield[A,S](a: A, s: S) extends Step[A,S]
  case class Skip[A,S](s: S) extends Step[A,S]

  /**
   * The stream class
   * @param A: the type of elements that the Stream sees
   */
  abstract class Stream[A]{ self =>

    /**
     * Type of the seed given to a stream
     */
    type S

    /**
     * The seed itself over which the stream "iterates"
     */
    def seed: S

    /**
     * The function that is used to step through the seed
     */
    def stepper(s: S): Step[A, S]
    ...
  }
}
{% endhighlight %}

Essentially, a stream is a function from a seed type `S` to a step `Step[A, S]`, which we call
`stepper`. The seed type, in our case is `List`. The `Step` data type serves three purposes:

  * yield an element.
  * skip an element. This is the ghost object I was referring to, as we will see a bit later.
  * be done. This signals that we will not be encountering any new elements.

For some intuition, here's a way to create a stream from a list:

{% highlight scala %}
/**
 * converting a list into a stream
 */
def toStream[T](ls: List[T]) = new Stream[T] {

  type S = List[T]
  def seed = ls

  def stepper(xs: List[T]) = xs match {
    case Nil => Done()
    case (y :: ys) => Yield(y, ys)
  }
}
{% endhighlight %}

This seems rather straightforward: if a list is empty, we are done, otherwise we yield the head of
the list, with the seed being the tail.

**Exercise 3**: Write functions `enumFromTo`, which creates a stream from a integer interval, and
`unStream`, which collapses a stream into a list:

{% highlight scala %}
def enumFromTo(a: Int, b: Int): Stream[Int] = ???

//inside class Stream[A]
def unStream: List[A] = ???
{% endhighlight %}

**Stream functions**


We can now safely attack stream functions. Let's do the simplest one first. Actually, it'll be
another exercise :-)

**Exercise 3**: Write the `map` function over streams:

{% highlight scala %}
//inside class Stream[A]
def map[B](f: A => B): Stream[B] = ???
{% endhighlight %}

For the `filter` function, the idea is to introduce an explicit `Skip` if the predicate is not
satisfied:

{% highlight scala %}
def filter(p: A => Boolean): Stream[A] = new Stream[A] {

  type S = self.S
  def seed = self.seed

  def stepper(s: S) = self.stepper(s) match {
    case Done() => Done()
    case Yield(a, s2) => if (p(a)) Yield(a, s2) else Skip(s2)
    case Skip(s) => Skip(s)
  }
}
{% endhighlight %}

Aha! So filtering is the place where we need to generate these ghost objects. Actually, the
advantage of this trick is very subtle. How would we have implemented filter otherwise? Well we'd
have to recursively call it. Intuitively, the problem there is that it breaks the railway track
analogy. We dearly want all stream operations to act as track operators.

Let's do `flatMap` now. Here again, it seems rather tricky to produce one element at a time. Indeed,
for every element `a: A` that we receive, we produce a `Stream[B]`. We first need to exhaust all
elements from this stream before going back to the top-level stream. The trick is to capture the
information that we are iterating over a particular (top-level or nested) stream as a piece of state:

{% highlight scala %}
def flatMap[B](f: A => Stream[B]) = new Stream[B] {

  /**
   * the seed is composed of the original seed from A
   * and a possible stream representing temporary results
   * from flattening
   */
  type S = (self.S, Option[Stream[B]])
  def seed = (self.seed, None)

  def stepper(s: S) = s match {

    /**
     * The case when we have either exhausted the
     * elements from `self`, or are in an intermediate
     * state, where a new `A` can appear
     */
    case (s1, None) => self.stepper(s1) match {
      case Done() => Done()
      case Yield(a, s2) => Skip((s2, Some(f(a))))
      case Skip(s2) => Skip(s2, None)
    }

    /**
     * The case when we are iterating through the stream
     * yielded by applying `f`
     */
    case (s1, Some(str)) => str.stepper(str.seed) match {
      case Done() => Skip((s1, None))
      case Yield(b, s2) =>

        val newStream = new Stream[B] {
          type S = str.S
          def seed = s2
          def stepper(s: S) = str.stepper(s)
        }

        Yield(b, (s1, Some(newStream)))

      case Skip(s2) =>
        val newStream = new Stream[B] {
          type S = str.S
          def seed = s2
          def stepper(s: S) = str.stepper(s)
        }

        Skip(s1, Some(newStream))
    }
  }
}
{% endhighlight %}

This state is captured in the seed type `S` for the resulting, flatmapped stream, which is a pair.
The right side of this pair corresponds to having a nested stream to run through.

Finally, let us also implement `zip`, which was difficult last week, but gets considerably easier
now:

{% highlight scala %}
def zip[B](b: Stream[B]) = new Stream[(A, B)] {

  type S = (self.S, b.S)
  def seed = (self.seed, b.seed)

  def stepper(s: S) = s match { case (sa, sb) =>

    (self.stepper(sa), b.stepper(sb)) match {
      case (Done(), Done()) => Done()
      case (Yield(a, sa2), Yield(b, sb2)) => Yield((a, b), (sa2, sb2))
      case (Yield(_, _), Skip(sb2)) => Skip((sa, sb2))
      case (Skip(sa2), Yield(_, _)) => Skip((sa2, sb))
    }
  }

}
{% endhighlight %}

Once again, we have a slightly more complex seed type. Indeed, both the left and right streams in
the `zip` could have skips. We want to synchronize arrivals on both streams in order to yield a
pair. If indeed we have a `Skip` on one side, we will just wait one more step.

**Exercise 4**: Verify that the `zip` function from last time can now be fused thanks to stream
fusion:

    zip(from(1, 10), from(2, 11))

Hint: you can use `enumFromTo` as the equivalent of `from` for lists.

The bottomline
--------------

The stream datastructure is a bit a hybrid push-pull structure. Although it primarily is a pull
structure (we request a new element given a state), because we can create ghost elements, we still
have the notion of proceeding one element at a time, which is the essence of a push abstraction.

I will end this article with a bit of intuition as to why it's easy to eliminate streams. As we said
before, streams are essentially functions from a seed to a stepper value. So they can be inlined at
call-site. Which means streams themselves don't exist. All we have is now functions that manipulate
pairs, `Option` and `Step`, which are algebraic data types. Haskell's compiler can for the most part
see these and eliminate them as well!

The explanation is very [wishiwashi](http://de.wikipedia.org/wiki/Wischiwaschi) admittedly. I hope
to explain how it works (maybe even work it out) soon enough though. So stay tuned!

The code
--------

You can find code related to this post [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/barbedwire/Stream.scala), and tests [here](https://github.com/manojo/functadelic/blob/master/src/test/scala/barbedwire/StreamSuite.scala).

References
----------

1. [Shortcut fusion for accumulating parameters & zip-like functions, Svenningsson, ICFP 2003][1]
2. [Theory and Practice of Fusion, Hinze et al., IFL 2010][2]
3. [Stream Fusion: from lists to streams to nothing at all, Coutts et al.
, ICFP 2007][3]

  [1]: http://dl.acm.org/citation.cfm?id=581491
  [2]: http://dl.acm.org/citation.cfm?id=2050135.2050137
  [3]: http://dl.acm.org/citation.cfm?id=1291199
