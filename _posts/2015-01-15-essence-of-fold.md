---
layout: post
title: "The Essence of Fold"
description: ""
category:
tags: []
---
{% include JB/setup %}

In the [last post]({% post_url 2015-01-11-lambda-calculus-direct-embedding %}), we saw that recursive
types are quite powerful. Using them it is possible to type any lambda expression. In this post, I will
pursue this exploration of recursive types a bit more. The exploration is focussed on what we can do
with these recursive types than type theory concepts related to them. We will however go a bit
[categorical](http://en.wikipedia.org/wiki/Category_theory). I will therefore assume some basic knowledge of category
theory (don't worry, I don't know much about it either, just enough to write this post).

Warmup: Basic Recursion
-----------------------

Let's do some simple exercises to get started with. We will have two running examples during the post,
[Church Numerals](http://en.wikipedia.org/wiki/Church_encoding), and lists of integers.

**Church Numerals**

Church Numerals are an encoding of the natural numbers. A number of type `Nat` is either defined as
zero, or as the successor of another number. In Scala, we can use case classes to represent numerals:

{% highlight scala %}
abstract class Nat
case object Zero extends Nat
case class Succ(n: Nat) extends Nat
{% endhighlight %}

**Exercise 1**: Write a function `nat2Int` that converts a Church numeral to its integer representation:

{% highlight scala %}
def nat2Int(n: Nat): Int = ???
{% endhighlight %}

**Integer Lists**

Another classic example is lists. A list of integers, `IntList` is either defined as the empty list
`Nil`, or a `Cons` list, which contains an integer, and an `IntList`, which is the tail.

{% highlight scala %}
abstract class IntList
case object Nil extends IntList
case class Cons(n: Int, xs: IntList) extends IntList
{% endhighlight %}

**Exercise 2**: Write a function `sumList` that sums up the elements of a list of integers:

{% highlight scala %}
def sumList(xs: IntList): Int = ???
{% endhighlight %}

Fix-Point on Types
------------------

Remember, we discussed [before]({% post_url 2014-09-03-understanding-the-fix-operator %}) how the
fixed-point combinator can be used to define recursive functions. In fact, it *lifts* the need for
native recursion. The combinator, called `y`, takes a function `f` as input, and satisfies the following property:

    y (f) = f (y (f))

Naturally, we can wonder whether there is an equivalent fixed-point combinator for types. Indeed, there is such a
combinator, which we call `Fix`. It's use is analogous to `y`. While `y` helps to succinctly define a function that
can unfold arbitrarily many times, `Fix` allows to succinctly specify a type than can be arbitrarily unfolded. Before
introducing `Fix`, let us first remove the recursion from the types we defined above in the examples.

**Church Numerals, non-recursive**

Take a look at the following definition:

{% highlight scala %}
abstract class NatF[+T]
case object Zero extends NatF[Nothing]
case class Succ[+T](t: T) extends NatF[T]
{% endhighlight %}

As we can see, `Succ` is now defined non-recursively.

**Exercise 3**: When we had `Nat`, the number `3` could be represented as `Succ(Succ(Succ(Zero)))`. How do we
represent `3` as a `NatF`? What is it's type?

**Integer lists, non-recursive**

Let's do the same with integer lists:

{% highlight scala %}
abstract class IntListF[+T]
case object NilF extends IntListF[Nothing]
case class ConsF[+T](n: Int, t: T) extends IntListF[T]
{% endhighlight %}

**Exercise 4**: What is the type of `ConsF(1, ConsF(2, ConsF(3, Zero)))`?

When in doubt, it is a good idea to use the console, try things out:

    scala> :t Succ(Succ(Succ(Zero)))
    Succ[Succ[Succ[Zero.type]]]

    scala> :t ConsF(1, ConsF(2, ConsF(3, NilF)))
    ConsF[ConsF[ConsF[NilF.type]]]

These examples show that the type for a particular value expands with that value. So a very large integer list,
or a very big Church numeral, would have a very big corresponding type. The intuition behind the `Fix` combinator
is to give a succinct type to any arbitrary sized value. Continuing the analogy, we expect `Fix` to satisfy the
following property:

    Fix[F] = F[Fix[F]]

**Exercise 5**: Replace `F` by `NatF` or `IntListF`. Does it make sense?

So let us create `Fix` in Scala. The first attempt

{% highlight scala %}
type Fix[F[_]] = F[Fix[F]]
{% endhighlight %}

does not work because we cannot define a type alias cyclically. Here is a second attempt:

{% highlight scala %}
case class Fix[F[_]](out: F[Fix[F]])
{% endhighlight %}

This works. It is either very subtle, or blindingly obvious, that a class definition as above
achieves two goals:

  * First, it defines a new type, `Fix[F[_]]`. The notation `F[_]` means that we expect `F` to
    be higher-kinded.
  * Second, it defines a function, or constructor. This function takes a value of type `F[Fix[F]]`, and
    returns a value of type `Fix[F]`. This indeed does satisfy the definition of a fixed-point combinator
    for types.

**Exercise 6**: Can we recreate `Nat` and `IntList` from `Fix`, `NatF`, `IntListF`? Hint: they can be
defined as type aliases.

Checkpoint
----------
This concludes the first section of this rather long post. We have seen that using `Fix` we can
create recursive types. So far it looks like we are just playing with types because we have some
visceral liking for it. Fear not, however. Our efforts will pay off soon. Before that, we will
have to take a bit of a diversion.

A categorical diversion
-----------------------

**Functors**

The suffix `F` in both `NatF` and `IntListF` stands for Functor. A functor is a concept in category theory,
that helps transform objects and arrows in a category C to objects and arrows in another category D,
respectively. This transformation has to satisfy a law which "preserves the structure" of the objects/arrows
that have been transformed. An endo-functor is a functor where the start category and the end category are the
same. Please do take a look at a more detailed definition if interested.

For the purpose of this post, we will work with a very specific category, which is the category **Scala**.
In this category, objects are types, and arrows are functions. What is a (endo-)functor for **Scala** then?
Well a functor `F` must:

  * take a type `T` to a type `F[T]`. For a bit of intuition, we can think of this as a function that
    lifts a value of type `T` to a value of type `F[T]`. Let's call this function `makeF`.
  * take a function `f: T => U` to a function `f_F: F[T] => F[U]`

The functor law that must be satisfied can be written as follows: for a given value `t: T`

    makeF(f(t)) === f_F(makeF(t))

As per this definition, we can see that `NatF` and `IntListF` already satisfy the first part. There is no
notion, however, of arrow transformation. An elegant way to do this is to create a special trait `Functor`
which forces the implementation of `f_F`, and provide implementations of `f_F` for both `NatF` and `IntListF`.
This is known as the type class approach. The `Functor` trait looks as follows:

{% highlight scala %}
trait Functor[F[_]] {
  /**
   * also known in categorical speak as Ff
   */
  def fmap[A, B](r: F[A], f: A => B): F[B]
}
{% endhighlight %}

Note that we change the name of `f_F` to `fmap`, which may ring a bell for some of you! Let us provide
evidences for our example functors:

{% highlight scala %}
/**
 * NatF is a (endo-)functor on the category of Scala types
 * It takes a type T to a type NatF[T]
 * it takes a function T => U to a function NatF[T] => NatF[U]
 * This is given by implementing an instance of the Functor typeclass
 */
object natFunctor extends Functor[NatF] {
  def fmap[A, B](r: NatF[A], f: A => B) = r match {
    case Zero => Zero
    case Succ(a) => Succ(f(a))
  }
}

/**
 * IntListF is a (endo-)functor on the category of Scala types
 * It takes a type T to a type IntListF[T]
 * it takes a function T => U to a function IntListF[T] => IntListF[U]
 * This is given by implementing an instance of the Functor typeclass
 */
object intListFunctor extends Functor[IntListF] {
  def fmap[A, B](r: IntListF[A], f: A => B) = r match {
    case NilF => NilF
    case ConsF(n, a) => ConsF(n, f(a))
  }
}
{% endhighlight %}

Understanding these implementations should be rather straightforward here. We pattern match on the
variants of each functor and apply the function as appropriate.

**Exercise 7**: Can you show that the functor law holds for the above implementations?

**F-Algebras**

So far, we have defined functors `NatF` and `IntListF`. It has been an interesting exercise, but
we cannot do anything with them just as of yet. We might, for instance, want to sum elements in a
list, or get the `Int` representation for a Church numeral (as in exercises 1 and 2). An F-Algebra
formalizes this concept. An F-Algebra is

  * an object `A` (from a category `C`).
  * a function `a: F[A] => A`.

In Scala, we can use represent an algebra as a type alias:

{% highlight scala %}
type Algebra[F[_], A] = F[A] => A
{% endhighlight %}

Here are example algebras, one for `NatF` and one for `IntListF`:

{% highlight scala %}
/**
 * An algebra for the NatF functor, that computes the successor function
 */
val intNatAlgebra: Algebra[NatF, Int] = (elem: NatF[Int]) => elem match {
  case Zero => 0
  case Succ(n) => n + 1
}

/**
 * An algebra for the IntListF functor, for summation
 */
val intListAlgebra: Algebra[IntListF, Int] = (elem: IntListF[Int]) => elem match {
  case NilF => 0
  case ConsF(n, x) => n + x
}
{% endhighlight %}

**Exercise 8**: Using `intNatAlgebra`, can you implement `nat2Int` from the first exercise?

Fold itself
-----------

Actually, the above question was given too early. We need a few more things to be able to confidently
answer it.

**The `Fix` Algebra**

We have defined algebras for our example functors, where the type `A` is `Int`. We can also create
algebras where `A` is the fix type:

{% highlight scala %}
type Nat = Fix[NatF]
val fixNatAlgebra: Algebra[NatF, Nat] = (elem: NatF[Nat]) => Fix(elem)

type IntList = Fix[IntListF]
val fixIntListAlgebra: Algebra[IntListF, IntList] =
  (elem: IntListF[IntList]) => Fix(elem)
{% endhighlight %}

Why do we even care for `Fix`? It's too mind-bending a type. Because

  1. We just reconstructed recursive types from our type system (aka, we did't use Scala's internal
     support for them).
  2. Because `Fix` forms a very particular algebra, known as the [initial algebra](http://en.wikipedia.org/wiki/Initial_algebra).
     From this algebra to any other algebra `(a, A)`, there is a `unique` mapping. The type of this unique mapping is

    uniqMap: (F[Fix[F]] => Fix[F]) => (F[A] => A)

Let's look at what `Fix[F]` means. It already represents the initial algebra *by itself*. As we saw above,
`Fix` is not only a type, it also represents a constructor, aka a function `F[Fix[F]] => Fix[F]`.

We can use the fact that this unique mapping exists. Given some algebra `F[A] => A`, we can implement
a function from to implement a function from `F[Fix[F]] => A`. Because, by definition, `F[Fix[F]] === Fix[F]`,
in fact we can implement a function from `Fix[F] => A`. This function is called `fold`.

{% highlight scala %}
def fold[F[_], A](alg: F[A] => A)(fx: Fix[F])(implicit ev: Functor[F]): A = {
  val lifted: F[Fix[F]] = fx.out
  val mapped: F[A] = ev.fmap(lifted, fold(alg) _)
  alg(mapped)
}
{% endhighlight %}

**Exercise 9**: Can you now implement, using `fold`, `Fix`, the functions `nat2List` and `sumList`?

The bottomline
--------------

We have discovered the true meaning of fold. It is one of the basic recursion schemes. It is a generic
way to evaluate recursive types!

## Related Readings

  * TAPL, chapter 20, explains recursive types in more detail.
  * A great FP-Complete [blog article](https://www.fpcomplete.com/user/bartosz/understanding-algebras)
    that give lots of intuition on the essence of evaluation.
  * A few articles by Debashish Ghosh which are great for understanding the above concepts. Check them out
    [here](http://debasishg.blogspot.ch/2012/01/list-algebras-and-fixpoint-combinator.html), [here](http://debasishg.blogspot.ch/2012/07/does-category-theory-make-you-better.html) and [here](http://debasishg.blogspot.ch/2011/07/datatype-generic-programming-in-scala.html)

