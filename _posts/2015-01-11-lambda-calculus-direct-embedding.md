---
layout: post
title: "A direct embedding of the untyped Lambda Calculus"
description: ""
category:
tags: []
---
{% include JB/setup %}

In the [last post]({% post_url 2014-09-03-understanding-the-fix-operator %}), I
tried to get an intuition for the fixed-point combinator. One aspect that I realised
(re-learned) on the way is that this combinator cannot be typed in the simply-typed
lambda calculus. We had to explicitly introduce the `fix` operator to our language
so that we could have recursion.

If we have a more powerful type system, we do not need an explicit operator,
however. The new feature we need is recursive types. The interesting thing is
that (as mentioned in TAPL 20) with recursive types, it is possible to properly
type any lambda expression. This also means that in a general purpose language
that has recursive types, we can create a *direct* embedding for the untyped
lambda calculus!

In this shortish post, I will try and explain how to achieve this embedding. Much
thanks to Prof. Viktor Kuncak whose example code I have been inspired by.

Direct embedding
----------------

There is a large amount of terminology in the world of domain specific languages
(DSLs). There are external and embedded DSLs. The latter mean that we use a host
language in which the DSL is implemented, and can be a form of library of. There
is a particular subcategory of embedded DSLs that is relevant to this post, a
*direct embedding* (for a more detailed discussion, please take a look at
Vojin's [paper](http://infoscience.epfl.ch/record/203432?ln=en)):

  * terms in the target language have a *direct* correspondence to terms in the
  host language. Primitive language constructs in the target language, such as
  conditional expressions, function literals and primitive operators, are mapped
  to their corresponding constructs in the host language.


The untyped lambda calculus
---------------------------

Before jumping in with the implementation, a quick reminder of the lambda calculus.
At it's core, the lambda calculus has 3 concepts: variables, abstraction, and
application. The beauty of the system is that any other programming language
concept (even booleans and numerals) can be encoded in this simple language. A
term in the lambda calculus has the following abstract syntax

    t ::= x        // variables
        | λx. t    // abstraction
        | t t      // application

For example, the `id` function, which takes an argument and returns it, is
written as `λx. x`.

A direct embedding
------------------

So how do we go about implementing the direct embedding in Scala? Remember, as
per the definition above, we do *not* want to represent lambda terms as data types.
Rather, we want to find a correspondence between variables, abstraction and
application in the lambda calculus and Scala features. And that leads us towards
the following intuition:

  * lambda abstraction corresponds to creating an anonymous function.
  * lambda application can basically be represented by function application in
  Scala. Recall that Scala is by default a strict language, so essentially we
  will be implementing a call-by-value evaluator.

Lambda application and abstraction must work in the lambda calculus domain. We can
use a special type `D` to represent this domain. As application must work, `D`
must have an `apply` function, that takes a value of type `D`, and returns another
value of type `D`. And therefore, we get the following for `D`:

{% highlight scala %}
abstract class D {
  def apply(d: D): D
}
{% endhighlight %}

For variables, we can lift values of the `Symbol` class to this domain. They are
special in that the `apply` function should not evaluate:

{% highlight scala %}
implicit def mkVar(s: Symbol) = new D {
  def apply(d: D) = throw new Error("does not evaluate")
}
{% endhighlight %}

For lambda abstraction, we follow our above intuition and get:

{% highlight scala %}
def lam(f: D => D) = new D {
  def apply(d: D) = f(d)
}
{% endhighlight %}

And that is essentially the extent of it!

The bottomline
--------------

The domain type `D` is an encoding of a recursive type. We have seen that with
such types, we can type any expression in the lambda calculus, by providing an
embedding for it.

The code
--------

You can find the code, with a few more examples like boolean and church numeral
encodings, [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/directlambda/LambdaCalculus.scala).
