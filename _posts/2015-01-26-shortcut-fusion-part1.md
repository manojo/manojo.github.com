---
layout: post
title: "Shortcut Fusion - Part 1"
description: ""
category:
tags: []
---
{% include JB/setup %}

Last [time]({% post_url 2015-01-15-essence-of-fold %}), we discovered that `fold` is one of the basic
recursion schemes. It is a generic way to evaluate recursive types. This high level of genericity is
very handy in practice: in this series of posts on fusion, I will try to show why.

Fusion, aka deforestation
-------------------------

In programming languages, fusion refers to the act of removing intermediate data structures in a
program. We want to do this so that less memory is allocated, and hopefully performance of the program
is improved as well. We concentrate in this series on lists. Here is an example of a program one might
want to fuse:

{% highlight scala %}
def dotproduct(xs: List[Int], ys: List[Int]): Int = {
  val zipped = xs zip ys
  val multiplied = zipped map { case (x, y) => x + y }
  multiplied.sum
}
{% endhighlight %}

The dot product function above is easy to read as it closely follows the mathematical definition.
We have created intermediate lists `zipped` and `multiplied`. In fact, we can write the same
function without creating any intermediate lists:

{% highlight scala %}
def dotproduct2(xs: List[Int], ys: List[Int]): Int = {
  def loop(xs2: List[Int], ys2: List[Int], tmp: Int): Int = (xs2, ys2) match {
    case (x :: rest1, y :: rest2) => loop(rest1, rest2, tmp + x * y)
    case (Nil, _) => tmp
    case (_, Nil) => tmp
  }

  loop(xs, ys, 0)
}
{% endhighlight %}

The function `dotproduct2` is arguably less easy to read. What saves it is a decent name for the
function. But it is more efficient. The idea behind fusion is to convert functions written in a
`dotproduct` style to efficient equivalents in the `dotproduct2` style, by "fusing"
operations together.

Moreover, we want a good fusion algorithm:

  1. The algorithm should work for as many different list operations as possible.
  2. The algorithm should be simple, elegant even.

And so with this in mind, let's have some fun with folds.

Fun with fold
-------------

Recall the fold function implementation over a `Fix` type from last week:

{% highlight scala %}
def fold[F[_], A](alg: F[A] => A)(fx: Fix[F])(implicit ev: Functor[F]): A = {
  val lifted: F[Fix[F]] = fx.out
  val mapped: F[A] = ev.fmap(lifted, fold(alg) _)
  alg(mapped)
}
{% endhighlight %}

Now that we know the theory, we can specialize this functions to lists only. This function is known
as `foldRight`:

{% highlight scala %}
def foldRight[A, B](z: B)(comb: (A, B) => B)(ls: List[A]): B = ls match {
  case Nil => z
  case x :: xs => comb(x, foldRight(z)(comb)(xs))
}
{% endhighlight %}

Going from the generic implementation, we now choose to use recursive (non-Fix) types, and inline
the functor's `fmap` implementation. As the name indicates, `foldRight` applies the function `comb` to
the right. So fold is one of the basic recursion schemes, which means that many functions on list can
be implemented using it. Here's the `map` function, which applies an function to each element of a
list:

{% highlight scala %}
def map[A, B](ls: List[A], f: A => B): List[B] = foldRight(List[B]()) {
  (x: A, acc: List[B]) => f(x) :: acc
}(ls)
{% endhighlight %}

**Exercise 1**: Implement `sum`, `filter`, `flatMap` using `foldRight`:

{% highlight scala %}
def sum(ls: List[Int]): Int = ???
def filter[A](ls: List[A], p: A => Boolean): List[A] = ???
def flatMap[A, B](ls: List[A], f: A => List[B]): List[B] = ???
{% endhighlight %}

There is also another way to fold over a list, namely `foldLeft`. Instead of collapsing elements to the
right, we collapse them to the left. Slightly trickier, but it turns out (what a surprise!) that it
can be implemented using `foldRight` as well.

**Exercise 2**: Implement `foldLeft` using `foldRight`:

{% highlight scala %}
def foldLeft[A, B](z: B)(f: (B, A) => B)(ls: List[A]): B = ???
{% endhighlight %}

To implement this function, we use a classic functional programming trick:

  * Try to make the types match.
  * When you can't, try creating an extra function abstraction, and later applying it.

It is easy to call `foldRight` directly on the list, but that messes us the evaluation order. We know
that `foldRight` proceeds right-to-left. The only way to proceed left-to-right, therefore, is to
*build a function* first, and then apply this function later. The function is built during the
right-to-left traversal. Let's call this function `inner`.

Naturally it's return type has to be `B`, because we expect it to be applied and to return the correct
result eventually. Let us think about the base case. If the input list `ls` is empty, the output must be `z`.
The result of `foldRight` should be a function, that, when given, something, returns `z`.

The combination function should accumulate the elements seen so far and create a function which will
eventually be use to accumulate in left-to-right order. This forces the type of `inner` to be `B => B`.
Hence the following implementation:

{% highlight scala %}
def foldLeft[A, B](z: B)(f: (B, A) => B)(ls: List[A]): B = {
  val inner = foldRight((x: B) => x) {
    (x: A, acc: B => B) =>
      (y: B) => acc(f(y, x))
  }(ls)
  inner(z)
}
{% endhighlight %}

Abstracting over list building
------------------------------

That was fun! It was also very useful. We were able to define list functions using `foldRight`. So
if we can find a good rule to fuse folds, then we have a simple fusion algorithm indeed. This is the
idea behind `foldr/build` fusion \[[1][1]\].

The name `foldr/build` suggests a presence of a dual operation. Indeed, folds can be viewed as operations
that consume/collapse lists, whereas a build operation constructs a list. Here is a signature of `build`
(I invite you to look at the paper for a curried version of this):

{% highlight scala %}
type build[A, B] = ((B, (A, B) => B)) => B) => B
{% endhighlight %}

This is a bit complicated to read, so let's break it down, and Scala-ify things. It looks like `build`
is a function that itself takes a function `g` and returns an element. The function `g` takes a pair and returns
an element. Let's inpect the pair a bit closer. The first element is an element of type `B`, and the
second is a function itself, that takes two elements, one of type `A`, the other of type `B`, and
returns an element of type `B`. But wait, we have seen this before!

**Exercise 3**: Replace `B` above by `List[A]`. What do you get? What if you replace `B` by `Int`?

That's right! The two elements are values for `Nil` and `Cons`. And if you remember from last week's
[post]({% post_url 2015-01-15-essence-of-fold %}), this is essentially a specialization of algebras
for the list functor! Let's rewrite this a bit more clearly:

{% highlight scala %}
trait ListAlgebra[A, B] {
  def nil: B
  def cons(a: A, b: B): B
}
def build[A, B](f: ListAlgebra[A, B] => B): B
{% endhighlight %}

We can create a specific instance for building lists, and a specific `build` function
that applies it:

{% highlight scala %}
class ListBuilder[A] extends ListAlgebra[A, List[A]] {
  def nil = Nil
  def cons(a: A, b: List[A]) = a :: b
}
def build[A](f: ListBuilder[A] => List[A]): List[A] = f(new ListBuilder[A])
{% endhighlight %}

We can now implement many list creating functions using `build`. Here's a more complicated one, for
zipping two lists:

{% highlight scala %}
def zip[A, B](xs: List[A], ys: List[B]): List[(A, B)] = {
  def zipBuilder = (b: ListBuilder[(A, B)]) => (xs, ys) match {
    case (x :: xs, y :: ys) => b.cons((x, y), zip(xs, ys))
    case _ => b.nil
  }
  build(zipBuilder)
}
{% endhighlight %}


**Exercise 4**: can you reimplement `map`, `flatMap`, and `filter` using buiders and foldR? What about `sum`?
**Exercise 5**: implement the `from` function, which creates a list of integers between a given bound,
using build:

{% highlight scala %}
def from(a: Int, b: Int): List[Int] = ???
{% endhighlight %}


The fusion part itself
----------------------

The fusion algorithm for `foldr/build` fusion is simple, yet elegant. It says that we can replace any
occurence of the following:

    foldRight(z)(comb)(build(gB))

with the following (pseudo-code):

    gB(new Builder(nil = z, cons = comb))

Anytime we are building a list and immediately consuming it, we can effectively get rid of the
intermediate list building. This is known as shortcut fusion because we introduce a rule that takes
a local short cut in terms of list building. In the rest of the article we will simple rewrite the above
as `gB(z, cons)`. The suffix `B` will denote the fact that we call a builder function.

Let's verify that this rule works for a sequence of two maps:

    map((map(xs, f)), g)
    ↪ build(mapB(map(xs, f), g)) //map defined using build
    ↪ build(foldR(mapBg.nil)(mapBg.cons)(map(xs, f)) //expanding mapB
    ↪ build(foldR(mapBg.nil)(mapBg.cons)(build(mapB(xs, f))) //map defined using build
    ↪ build(mapB(xs, f)(mapBg.nil, mapBg.cons)) //using foldr/build rule

As we have seen, `mapB` is itself defined using `foldRight`. The combination function for this `foldRight`
will be the composition of `f` and `g` (working a few more steps), and we notice that indeed, we have been
rid of the intermediate list!

**Exercise 6**: verify that we get rid of all intermediate lists in the `dotproduct` function from
above, if `map` and `zip` are defined using `build` and `foldRight`.

Caveats
-------

Though it looks wonderful, there are a few fusion cases that are not very well covered by `foldr/build`
fusion. The best example is that of zipping two lists. While we can get rid of the list produced by `zip`,
as we saw in the `dotproduct` case, zipping the input lists is not possible with the current algorithm.

Let's see why. Here's the simplest example I can think of:

{% highlight scala %}
zip(from(1, 10), from(2, 11))
{% endhighlight %}

This evaluates to:

    zip(from(1, 10), from(2, 11))
    ↪ build(zipB(from(1, 10), from(2, 11))) //zip defined using build
    ↪ build(zipB(build(fromB(1, 10)), build(fromB(2, 11))) //from defined using build
    ↪ ???

There is no rule we can apply to simplify the above. So we cannot apply the `foldr/build` fusion
rule here. But, what if we could define `zip` using `foldRight`?

**Exercise 7**: Implement a function `zip2` using `foldRight`:

{% highlight scala %}
def zip2[A, B](xs: List[A], ys: List[B]): List[(A, B)] = ???
{% endhighlight %}

This is another tricky one. So let's resort to the classic functional programming trick again. We
get the following solution:

{% highlight scala %}
def zip2[A, B](xs: List[A], ys: List[B]) = {
  val done = (zs: List[B]) => List[(A, B)]()

  val comp = (x: A, acc: List[B] => List[(A, B)]) => {
    zs: List[B] => zs match {
      case Nil => Nil
      case z :: zs2 => (x, z) :: acc(zs2)
    }
  }

  (foldRight(done)(comp)(xs)) (ys)
}
{% endhighlight %}

Indeed, we can use `foldRight` on a single list. So we have to build a function in the first pass which,
when applied construct the list of pairs. Using this new definition, we get the following evaluation
for our example

    zip2(from(1, 10), from(2, 11))
    ↪ zip2(build(fromB(1, 10)), build(fromB(2, 11))) // from defined using build
    ↪ zip2(xs, ys) // renaming for simplicity
    ↪ (foldR(z)(comb)(xs))(ys) // zip2 defined using foldRight
    ↪ (foldR(z)(comb)(build(fromB(1, 10))))(ys)
    ↪ (fromB(1, 10)(z)(comb))(ys)

We can definitely get rid of the first intermediate list, but not the second one. How could we get rid
of both input lists? This is a job for the next posts in the series!

The bottomline
--------------

Folds are a lot of fun. We can implement a lot of list functions using folds. They also turn out to
be useful in practice (who knew!). If we abstract over list building as well, then we can use a single
rule to fuse operations together, and get rid of intermediate lists. The approach is not complete
however, but we'll soon see how we can improve upon it.

References
----------


1. [A shortcut to deforestation, Gill et al., FPCA 1993][1]
2. [Theory and Practice of Fusion, Hinze et al., IFL 2010][2]

  [1]: http://dl.acm.org/citation.cfm?id=165214
  [2]: http://dl.acm.org/citation.cfm?id=2050135.2050137
