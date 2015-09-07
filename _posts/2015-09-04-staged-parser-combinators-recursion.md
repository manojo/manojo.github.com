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
lazy val fib: Int => Int = memo { i =>
  if (i == 0 || i == 1) 1
  else fib(i - 1) + fib(i - 2)
}
{% endhighlight %}

Indeed, thanks to laziness, the first time we use `fib` a reference for it will
be created and then properly bound to further uses of `fib`. As a result any
internal structures that `memo` uses will be bound to that reference. And that
should definitely give you enough info to implement `memo` now!


Recursion and Staging
---------------------

So what is the problem with recursion and staging? Seems that it should work out
of the box? Well, consider the following recursive, staged parser, which parses
digits and adds them up:

{% highlight scala %}
def adder: Parser[Int] = (digit2Int ~ adder) map { case (x, y) => x + y } | digit2Int
{% endhighlight %}

In a staged setting, `adder` is a code generator for a parser. So every time we
encounter it we generate code for it. Being recursive, this process will last forever,
theoretically. In practice we get stack overflows. Neither are desirable, of course.

For recursive parsers, we need to generate two different types of code for the same
parser:

  * A function when we encounter the parser for the first time.
  * A call to the generated function every other time.

We need to remember whether we have encountered a recursive parser, and we can use
memo-ization for that! Just like the `memo` function above, we define a `rec`
combinator which wraps a recursive parser. It is similar to the [fix combinator](http://localhost:4000/2014/09/03/understanding-the-fix-operator/).
Our example now becomes:

{% highlight scala %}
lazy val adder: Parser[Int] = rec {
  (digit2Int ~ adder) map { case (x, y) => x + y } | digit2Int
}
{% endhighlight %}

One can see the use of lazy vals as a drawback, but it turns out that it is a
common pattern even for standard parsers. For example packrat parsers need to be
declared as lazy vals to function correctly \[[1][1]\].

The implementation of `rec` turns out to be very close to that of `memo`:

{% highlight scala %}
trait StagedParsersExp
    extends StagedParsers
    ...
    with ParseResultOpsExp
    with FunctionsExp {

  val store = new scala.collection.mutable.HashMap[Parser[_], Sym[_]]

  def rec[T: Typ](p: Parser[T]) = Parser[T] { in =>

    import Parser._

    val myFun: Rep[Input => ParseResult[T]] = store.get(p) match {

      case Some(f) =>
        scala.Console.println("we have a function call")
        val realf = f.asInstanceOf[Exp[Input => ParseResult[T]]]
        realf

      case None =>
        scala.Console.println("first time we see this guy, creating a new symbol")

        val funSym = fresh[Input => ParseResult[T]]

        store += (p -> funSym)
        val f = p.toParseResult
        val g = createDefinition(funSym, doLambdaDef(f))
        store -= p

        funSym
    }

    val res: Rep[ParseResult[T]] = myFun(in)
    conditional(res.isEmpty,
      ParseResultCPS.Failure(res.next),
      ParseResultCPS.Success(res.get, res.next)
    )
  }
}
{% endhighlight %}

Some important points:

  * Our cache (or store) maps `Parser` instances to LMS IR symbols. The idea is
  to generate a fresh function symbol the first time we see a parser, and bind
  the generated function to it.
  * The function we generate is of type `Rep[Input => ParseResult]`, i.e. it is
  a staged function (exactly what we need). We also use a struct representation
  for parse results, as functions also act as join points.
  * The `doLambdaDef` function reifies an unstaged function: it converts a
  `Rep[T] => Rep[U]` to a `Rep[T => U]`, and is available my mixing in the `Functions`
  traits.
  * The `createDefinition` function is also LMS IR code that binds a symbol to a
  `Rep`.


The bottomline
--------------

So that's it! We have now added recursion to our staged parser combinators. Before
we conclude, there are a few points worth discussing.

__Alternatives to staging recursion__

It might be a bit disappointing that we have to use an explicit `rec` combinator
because the information about recursion seems directly available anyway. A possibility
would be to use macros or reflection to inspect the structure of a parser, and add
the `rec` combinator at compile time. But if we use macros, we could directly do
partial evaluation with them, and just leave recursive functions untouched. This
is what [Eric](https://github.com/begeric) does in his parser combinators and
macros work\[[2][2]\].

__A note about laziness__

In this post we saw that lazy vals are very useful for late binding of recursive
constructs. This is another classic functional programming trick. In a language
like Haskell, because everything is lazy by default we don't realize it as much,
but in Scala we are required to actively think about it.

We also need to be a bit careful with definitions of our combinators, in order
to propage the laziness correctly. This is why, for example, the sequencing combinator
`~` takes a by-name parameter:

{% highlight scala %}
def ~[U: Typ](that: => Parser[U]): Parser[(T, U)]
{% endhighlight %}

If it didn't, the value of the right hand side would be computed when the expression
is created, and in a recursive case this blows up.


The code
--------

The code used in this post can be accessed through the following files:

  * The source implementation [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/stagedparsec/StagedParsers.scala#L148).
  * A test file [here](https://github.com/manojo/functadelic/blob/master/src/test/scala/stagedparsec/RecParsersSuite.scala).
  * A check file [here](https://github.com/manojo/functadelic/blob/master/test-out/rec-parser.check).

If you want to run this code locally, please follow instructions for setup on
the readme [here](https://github.com/manojo/functadelic). The `sbt` command to
run the test is `test-only stagedparsec.RecParsersSuite`


References
----------

1. [Packrat Parsing in Scala, Jonnalagedda., Tech. Report 2009][1]
2. [Accelerating Parser Combinators with Macros, Béguet et al., Scala 2014][2].
3. [Staged Parser Combinators for Efficient Data Processing, Jonnalagedda et al., OOPSLA 2014][3]

  [1]: http://scala-programming-language.1934581.n4.nabble.com/attachment/1956909/0/packrat_parsers.pdf
  [2]: http://infoscience.epfl.ch/record/200905?ln=en
  [3]: http://infoscience.epfl.ch/record/203076?ln=en
