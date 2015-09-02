---
layout: post
title: "Staged Parser Combinators"
description: ""
category:
tags: []
---
{% include JB/setup %}

We previously [observed]({% post_url 2015-02-19-staging-foldleft %}), a long time
ago, that combining partial evaluation and CPS-encoded data structures enabled us
to achieve fusion. To be a bit elaborate:

  1. If we CPS-encode data structures, fusion on these structures reduces to fusion
  on function composition.
  2. By partially evaluating function composition we effectively achieve fusion.

In this post, we will explore another interesting application of this principle,
parser combinators \[[1][1]\]. Essentially, through the use of staging, we will
turn parser combinators into parser generators. The latter are rid of the composition
overhead of combinators, and therefore have good runtime performance \[[2][2]\].

This post heavily borrows from the lessons learnt on the previous adventure with
fold-based fusion. I will therefore assume that the reader is familiar with the
staging techniques we have used so far. I'd recommend reading the paragraph on
LMS over [here]({% post_url 2015-02-19-staging-foldleft %}) in order to get an idea.

Writing Parsers
---------------

__What is a parser?__

A parser is a program that reads some input (usually text) and converts it into
a structured representation of this input. For example, suppose you are given a
text file containing some information about tennis players, as follows:

    firstname: "Roger", lastname: "Federer", grandslams: 18, currentranking: 2
    firstname: "Rafael", lastname: "Nadal", grandslams: 14, currentranking: 8
    firstname: "Novak", lastname: "Djokovic", grandslams: 8, currentranking: 1

If we want to operate on this data, we must first read it and convert it into a
format that a program can handle. This is where a parser comes in. A possible
conversion of the above data, in Scala, would be to a list of case classes that
represent players, as follows:

{% highlight scala %}
case class TennisPlayer(
  firstname: String,
  lastname: String,
  grandSlams: Int,
  currentRanking: Int)

parse(aTextFile)

//yields the following result
List(
 TennisPlayer("Roger", "Federer", 18, 2),
 TennisPlayer("Rafael", "Nadal", 14, 4),
 TennisPlayer("Novak", "Djokovic", 18, 1)
)
{% endhighlight %}

Another classic, although meta, example of a parser is a program that reads another
program, and converts into a tree format.

__How to write a parser__

There are three traditional ways of writing parsers:

  1. Write a parser "by hand". For the above example, that amounts to the following
  pseudocode sequence:
      - While a text file is not empty, do the following:
      - read a line, separated by commas.
      - at each line, read key-value pairs corresponding to attributes of a player.

  2. Use a parser generator. You may have noticed that even the text file above
  respects a structured formatting. In other words, it adheres to a certain grammar.
  Therefore an elegant way to write a parser is to write the grammar that corresponds
  to the structured format, and generate the "by hand" version. There are many
  popular parser generators available. Such as yacc, ANTLR, Happy.

  3. Use parser combinators, the topic of this post. Parser combinators are a library
  approach to writing parsers. The idea is similar to parser generators, i.e. to
  write grammars. These grammars are themselves _programs_. Complex grammars
  are built from simpler ones through the use of combinator functions.

Grammars and parsers do not always have a one-to-one correspondence. Some grammars
are more powerful/general than others, and some parsing techniques are better suited
to some grammars. In this post we consider those grammars that can be parsed in
a left-to-right, recursive-descent manner.

Parser Combinators
------------------

At its heart, a parser is a function that takes an input state and produces a result.
For simplicity, we consider the input to be an array of characters. The input state
also contains some information regarding where in the text we currently are. Here
is a possible type alias for a parser that parsers elements of type `T`:

{% highlight scala %}
type Input
type Parser[T] = Input => ParseResult[T]
{% endhighlight %}

Before we go further, we need to take a quick look at what `Input` and `ParseResult`
are.

__Input__

At its core, and input state knows whether it is at the end of the input, how to
get the current element, and access the remaining input (excluding the current
element). Typically, the above API is known to be a `Reader`:

{% highlight scala %}
abstract class Reader[+T] {
  def atEnd: Boolean
  def first: T
  def rest: Reader[T]
}
{% endhighlight %}

As we are working with arrays of characters as input sources, our base element is
a `Char`.

__Exercise 1__: Implement the `Reader` API for a `StringReader` :

{% highlight scala %}
class StringReader(source: Array[Char], offset: Int = 0) extends Reader[Char] {
  def atEnd: Boolean = ???
  def first: Char = ???
  def rest: Reader[Char] = ???
}
{% endhighlight %}

__ParseResult__

A `ParseResult`, as its name suggests, encapsulates the result of parsing an input.
It can either be a succesful parse, in which case we have a result, or a failure,
in which case we have some basic information as to where the parsing failed. Many
parser combinator frameworks add different types of failures and extra information,
such as error messages to make the library more user-friendly. We will focus on
the basics here:

{% highlight scala %}
abstract class ParseResult[+T] {
  val next: Input
  def isEmpty: Boolean
}
case class Success[+T](res: T, override val next: Input) extends ParseResult[T] {
  def isEmpty = false
}
case class Failure(override val next: Input) extends ParseResult[Nothing] {
  def isEmpty = true
}
{% endhighlight %}

__Exercise 2__: Implement some usual suspects for `ParseResult`. You can use
object-oriented style or pattern-matching:

{% highlight scala %}
abstract class ParseResult[+T] {
  ...
  def map[U](f: T => U): ParseResult[U] = ???
  def orElse[U >: T](that: => ParseResult[U]): ParseResult[U] = ???
}
{% endhighlight %}

__Elementary Parser Combinators__

Now that we understand what `Input` and `ParseResult` are, we can build our first
parser, which succeeds if an element satisfies a certain predicate and fails if
it does not. We take a similar design route to the one we took
[in the past](http://manojo.github.io/2015/02/19/staging-foldleft):

  - Parsers live in a `Parsers` trait, which acts as the context.
  - A `Parser` extends a function
  - The `Parser` companion object defines an `apply` method which acts as
  syntactic sugar.

Packed together, we get the following:

{% highlight scala %}
trait Parsers {

  /** a type alias for a single element a Reader can read */
  type Elem
  type Input = Reader[Elem]

  abstract class Parser[+T] extends (Input => ParseResult[T]) { ... }

  object Parser {
    def apply[+T](f: Input => ParseResult[T]) = new Parser[T] {
      def apply(in: Input) = f(in)
    }
  }

  def acceptIf(p: Elem => Boolean) = Parser[Elem] { in =>
    if (in.atEnd) Failure(in)
    else if (p(in.first)) Success(in.first, in.rest)
    else Failure(in)
  }

  def accept(e: Elem) = acceptIf(_ == e)
}
{% endhighlight %}

So `acceptIf` is the most basic parser we can build. We of course need to first
make sure that we haven't reached the end of the input.

__Exercise 3__: Implement some more elementary parsers that parse single characters,
namely `digit` and `letter`, which successfully parse digits and letters. You are
only allowed to use `acceptIf`. Then implement `digit2Int`, which converts a parsed
digit into its integer representation.

{% highlight scala %}
trait CharParsers extends Parsers {
  type Elem = Char

  def digit: Parser[Char] = ???
  def letter: Parser[Char] = ???
  def digit2Int: Parser[Int] = ???

}
{% endhighlight %}

Note that we have in the `CharParsers` trait, we know the elements we read to be
characters.

__The Parser API__

It turns out that parser combinators are monads \[[3][3]\]! Apart from making us
categorically happy, this means we can once again implement some usual suspects.

__Exercise 4__: Implement `map` and `flatMap` for `Parser`. You may want, for
simplicity sake, implement a method `flatMapWithNext` for `ParseResult` first:

{% highlight scala %}
abstract class ParseResult[+T] {
  def flatMapWithNext[U](f: T => Input => ParseResult[U]): ParseResult[U] = ???
}

abstract class Parser[+T] extends (Input => ParseResult[T]) {
  def map[U](f: T => U): Parser[U] = ???
  def flatMap[U](f: T => Parser[U]): Parser[U] = ???
}
{% endhighlight %}

The solution really lies in implementing the corresponding functions for
`ParseResult`. We want `flatMapWithNext` to propagate the failure state, but
apply the function to its result otherwise:

{% highlight scala %}
def flatMapWithNext[U](f: T => Input => ParseResult[U]) = this match {
  case Success(res, next) => f(res)(next)
  case fail @ Failure(_) => fail
}
{% endhighlight %}

The Parser API then becomes obvious:

{% highlight scala %}
def map[U](f: T => U): Parser[U] = Parser { in => this(in) map f }
def flatMap[U](f: T => Parser[U]): Parser[U] = Parser { in =>
  this(in) flatMapWithNext f
}
{% endhighlight %}


__Sequencing and Alternation__

We finally come to the meat of parser combinators, sequencing and alternation:

 * The sequencing combinator `~` takes two parsers `l`, `r`, and creates
 a new parser that succeeds if `l` succeeds, and then `r` succeeds on the
 remaining input. The result is a pair containing results of both `l` and `r`.
 There are additional variants of the combinator, `~>` and `<~`, which respectively
 discard the left and the right part after successfully parsing both of them.

 * The alternation combinator `|`takes two parsers `l`, `r`, and succeeds if
 either `l` or `r` succeed. It first tries `l` and tries `r` only if `l` fails.
 This is as per the [PEG notation for grammars](https://en.wikipedia.org/wiki/Parsing_expression_grammar).

__Exercise 5__: Given the above specs, implement `~` and its cousins, as well as
`|`. Hint: you've already done most of the work before, and can implement sequencing
using for-comprehensions only.

{% highlight scala %}
abstract class Parser[+T] extends (Input => ParseResult[T]) {
  ...
  def ~[U](that: => Parser[U]): Parser[(T, U)] = ???
  def ~>[U](that: => Parser[U]): Parser[U] = ???
  def <~[U](that: => Parser[U]): Parser[T] = ???
  def |[U >: T](that: => Parser[U]): Parser[U] = ???
}
{% endhighlight %}


Staging Parser Combinators
--------------------------

So there we are, we built the basics of a parser combinator library. By now, I guess
you would have realised that with bigger parsers are created from elementary ones
through function composition. This is because parsers are represented as functions.
Which means that if we partially evaluate function composition we get rid of a lot
of overhead!

And so we use our favorite staging framework, LMS, to achieve this. The trick, as
usual, is to use the `Rep` type judiciously.

__Exercise 6__: Given the original API for `Parser` below, add `Rep` types
where required.

{% highlight scala %}
abstract class Parser[+T] extends (Input => ParseResult[T]) {
  def map[U](f: T => U): Parser[U]
  def flatMap[U](f: T => Parser[U]): Parser[U]

  def ~[U](that: => Parser[U]): Parser[(T, U)]
  def ~>[U](that: => Parser[U]): Parser[U]
  def <~[U](that: => Parser[U]): Parser[T]

  def |[U >: T](that: => Parser[U]): Parser[U]

}
{% endhighlight %}

If you have followed previous blogposts, you will find this rather easy: as we
want to partially evaluate function composition, all functions are 'non-Repped',
while input and output types are 'Repped':

{% highlight scala %}
abstract class Parser[T: Typ] extends (Rep[Input] => Rep[ParseResult[T]]) {
  def map[U: Typ](f: Rep[T] => Rep[U]): Parser[U]
  def flatMap[U: Typ](f: Rep[T] => Parser[U]): Parser[U]

  def ~[U: Typ](that: => Parser[U]): Parser[(T, U)]
  def ~>[U: Typ](that: => Parser[U]): Parser[U]
  def <~[U: Typ](that: => Parser[U]): Parser[T]

  def |[T: Typ](that: => Parser[U]): Parser[U]

}
{% endhighlight %}

Once again, the tricky method to type correctly above could be `flatMap`. Once
again, remembering that `Parser` is now a code generator helps ease all concerns.
The _implementation_ of the methods themselves does not change much, as we mostly
delegate the logic to underlying types.

___Note___: for those who have been following the posts, you may have noticed that the
`Manifest` typeclass is now replaced by the `Typ` typeclass. This does not fundamentally
affect the behaviour of the partial evaluation engine that we use, but if you are
interested you can still read all about it
[here](https://groups.google.com/forum/#!topic/delite-devel/OnSdvryMWwY).

__CPS encoding ParseResult__

To achieve fold-based fusion before, we first had to come up with a late representation
of lists, in the form of the fold function. Only after that could we partially evaluate
function composition, and get fusion.

For parser combinators, we already start with a function representation for parsers.
Yet we can still benefit from our good old trick of CPS-encoding data structures,
similarly to what we did with the [`Either` type](http://manojo.github.io/2015/03/20/cps-encoding-either/).
Indeed, we still create quite a few intermediate structures, for every single parser:
we box the `Input` after every parsing attempt, and we create a `ParseResult` structure
at the same time.

These can result in considerable performance overhead. At the moment (day of writing),
I do not know exactly how considerable, and therefore whether we should really
invest effort in deforesting these. Moreover, if we generate C code, for instance,
we can generate stack-allocated structs which are consider zero overhead. In Scala
we could generate [value classes](http://docs.scala-lang.org/overviews/core/value-classes.html).

Nonetheless, it is a good exercise to try it, because:

  - CPS encodings and partial evaluation mix well, we get deforestation as a library.
  - We will see below that we gain some insights about join points and code generation.

And so let's [just do it](https://youtu.be/DvVUBZy_MHE?t=8m55s), for `ParseResult`
at least.

__Exercise 7__: Can you find a CPS-encoding for `ParseResult`?

Yes you can! A `ParseResult` is either a success or a failure, so it's but a glorified
instance of the `Option` type, which itself is a basic version of the `Either` type.
Here is a math-ish notation:

    type ParseResultCPS[T] = forall X. ((T, Input) => X, Input => X) => X

Adding `Rep` types as appropriate, we get the following:

{% highlight scala %}
abstract class ParseResultCPS[T: Typ] { self =>

  def apply[X: Typ](
    success: (Rep[T], Rep[Input]) => Rep[X],
    failure: Rep[Input] => Rep[X]
  ): Rep[X]

  ...
}

object ParseResultCPS {
  def Success[T: Typ](t: Rep[T], next: Rep[Input]) = new ParseResultCPS[T] {
    def apply[X: Typ](
      success: (Rep[T], Rep[Input]) => Rep[X],
      failure: (Rep[Input]) => Rep[X]
    ): Rep[X] = success(t, next)
  }

  def Failure[T: Typ](next: Rep[Input]) = new ParseResultCPS[T] {
    def apply[X: Typ](
      success: (Rep[T], Rep[Input]) => Rep[X],
      failure: (Rep[Input]) => Rep[X]
    ): Rep[X] = failure(next)
  }
}

{% endhighlight %}

We define `Success` and `Failure` functions for the sake of a friendly API. They
respectively apply the `success` and `failure` continuations.

__Exercise 8__: Implement `map`, `flatMapWithNext`, and `orElse` for `ParseResultCPS`.

Let me give you the implementation for the second one:

{% highlight scala %}
def flatMapWithNext[U: Typ](f: (Rep[T], Rep[Input]) => ParseResultCPS[U])
    = new ParseResultCPS[U] {

  def apply[X: Typ](
    success: (Rep[U], Rep[Input]) => Rep[X],
    failure: Rep[Input] => Rep[X]
  ): Rep[X] = self.apply(
    (t: Rep[T], in: Rep[Input]) => f(t, in).apply(success, failure),
    failure
  )
}
{% endhighlight %}

Note that we have slightly changed the signature of the function `f`, but [Mr.
Schoenfinkel](https://en.wikipedia.org/wiki/Currying) tells us that both forms
are equivalent.

As a result, the signature of `Parser` now becomes the following:


{% highlight scala %}
abstract class Parser[T: Typ] extends (Rep[Input] => ParseResultCPS[T])
{% endhighlight %}

Of course, we eventually need to materialize our result, i.e. call the `apply`
function on an instance of `ParseResultCPS`. This is analogous to our need to
eventually materialize a staged `FoldLeft`, either as a list, or as another
aggregate (for example `Int` if we were summing elements). We'll define a handy
`toOption` function, which returns a `Rep[Option]`:

{% highlight scala %}
trait ParseResultCPS extends ...
    with LiftVariables
    with OptionOps
    with ZeroVal {

  abstract class ParseResultCPS[T: Typ] { self =>
    ...
    def toOption: Rep[Option[T]] = {
      var isEmpty = unit(true); var value = zeroVal[T]
      self.apply(
        (t, _) => { isEmpty = unit(false); value = t },
        _ => unit(())
      )
      if (isEmpty) none[T]() else make_opt(Some(readVar(value)))
    }
  }

}
{% endhighlight %}

Note that we can use variables by mixing in the `LiftVariables` trait.

Join Points and Code Generation
-------------------------------

We have one final issue to take care of. We saw it once [a while ago](http://manojo.github.io/2015/04/17/staged-foldleft-improvements/),
but it's  worth taking a more in-depth look.

__Exercise 9__: What does the following conditional expression generate?

    (if (c1) Success(t1, in1) else Success(t2, in2)).flatMapWithNext(someF).apply(success, failure)

This is the evaluation trail:

    (if (c1) Success(t1, in1) else Success(t2, in2)).flatMapWithNext(someF).longPipeline.apply(success, failure)
    ↪ (if (c1) Success(t1, in1).flatMapWithNext(someF).longPipeline
       else    Success(t2, in2).flatMapWithNext(someF).longPipeline).apply(success, failure)
    ↪ if (c1) Success(t1, in1).flatMapWithNext(someF).longPipeline.apply(success, failure)
      else    Success(t2, in2).flatMapWithNext(someF).longPipeline.apply(success, failure)

Now if `someF` also introduces a conditional expression, then `longPipeline` gets
generated in those branches as well. This leads to code explosion, exponential in
the number of conditionals. But does such code occur in real life?

__Exercise 10__: Can you find an example of a parser which has a shape similar to the conditional
expression above?

Here is a possible answer to that question:

{% highlight scala %}
val parser = (t1 | t2) ~ someF ~ longPipeLine
{% endhighlight %}

A perfectly normal-looking parser indeed. The crux of the problem lies in the fact
that conditional expressions act split points. If we partially evaluate function
composition naively, code gets generated in both branches. We therefore need to
create deliberate join points. In a non CPS-encoded representation of parse results,
the boxing acts as a join point. We can instead introduce temporary variables for
the relevant fields of a parse result. These variables will be affected by the use
of the `success` and `failure` continuations. After that, they are effectively
passed on further, thereby acting as join points.

For this, we special case the construction of a `ParseResultCPS` with a conditional
expression:

{% highlight scala %}
case class ParseResultCPSCond[T: Typ](
  cond: Rep[Boolean],
  t: ParseResultCPS[T],
  e: ParseResultCPS[T]
) extends ParseResultCPS[T] { self =>

  def apply[X: Typ](
    success: (Rep[T], Rep[Input]) => Rep[X],
    failure: Rep[Input] => Rep[X]
  ): Rep[X] = if (cond) t(success, failure) else e(success, failure)

  ...
}
{% endhighlight %}

The `apply` function is implemented in the naive way, but we need to override all
other methods of the API to use temporary variables. I will just show map here,
I am sure you can figure out the rest of them!

{% highlight scala %}
case class ParseResultCPSCond[T: Typ](...){

  override def map[U: Typ](f: Rep[T] => Rep[U]) = new ParseResultCPS[U] {
    def apply[X: Typ](
      success: (Rep[U], Rep[Input]) => Rep[X],
      failure: Rep[Input] => Rep[X]
    ): Rep[X] = {
      var isEmpty = unit(true); var value = zeroVal[T]; var rdr = zeroVal[Input]

      self.apply[Unit](
        (x, next) => { isEmpty = unit(false); value = x; rdr = next },
        next => rdr = next
      )

      if (isEmpty) failure(rdr) else success(f(value), rdr)
    }
  }


}
{% endhighlight %}

As you can see, we first evaluate `self`, which is the pipeline before the call
to `map`, by passing trivial continuations that assign results to variables. The
`success` and `failure` continuations are called on these temporary results.


The bottomline
--------------

We have seen in this post what parser combinators are, and how to stage them. We
also saw how to deforest parse results, through the use of our good friend CPS-encoding.
For conditional expressions, we realised the need to create join points. And we
did all this by staying true to our principle of only using libary-style coding.

In the coming posts, we will add a few more features to this staged combinator
library, so stay tuned!

_Note_: Much of the work in this post has been previously documented in a paper
by my colleagues and me \[[2][2]]. This post acts as a revival of that work along
with some cleanup and fresh ideas (ex. CPS-encoding ParseResult). If you have been
through that paper, you will notice that we use a different trick for alternation,
namely the introduction of functions (Section 3.3). Turns out that all we need is
some temporary variables, as seen above, so you can conveniently ignore that section.


The code
--------

The code used in this post can be accessed through the following files:

  * The source implementation [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/stagedparsec/StagedParsers.scala).
  * An implementation of char-based parsers [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/stagedparsec/CharParsers.scala)
  * The implementation for `ParseResultCPS` [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/stagedparsec/ParseResultCPS.scala).
  * The implementation of `Reader` [here](https://github.com/manojo/functadelic/blob/master/src/main/scala/stagedparsec/StructReaderOps.scala).
  As you will see, it uses the struct representation available in LMS.
  * If you dig into the lms.util folder you can find other CPS encodings, because why not.

  * Test files:
    * [Parsers](https://github.com/manojo/functadelic/blob/master/src/test/scala/stagedparsec/CharParsersSuite.scala).
    * [ParseResultCPS](https://github.com/manojo/functadelic/blob/master/src/test/scala/stagedparsec/ParseResultCPSSuite.scala).
    In particular you might be interested in `flatMapTilde`, `orElseAlternation`, `orElseSeq`.
    * [Reader](https://github.com/manojo/functadelic/blob/master/src/test/scala/stagedparsec/StringReaderSuite.scala).

  * Check files that contains generated code:
    * [Parsers](https://github.com/manojo/functadelic/blob/master/test-out/char-parser.check).
    * [ParseResultCPS](https://github.com/manojo/functadelic/blob/master/test-out/parseresultcps.check).
    * [Reader](https://github.com/manojo/functadelic/blob/master/test-out/stringreader.check).

If you want to run this code locally, please follow instructions for setup on
the readme [here](https://github.com/manojo/functadelic). The `sbt` commands to
run the particular tests are

  * `test-only stagedparsec.CharParsersSuite`
  * `test-only stagedparsec.ParseResultCPSSuite`
  * `test-only stagedparsec.StringReaderSuite`


References
----------

1. [Direct style monadic parser combinators for the real world, Leijen et al., Tech. Report 2001][1]
2. [Staged Parser Combinators for Efficient Data Processing, Jonnalagedda et al., OOPSLA 2014][2]
3. [Monads for Functional Programming, Wadler, 1995][3]
4. [Accelerating Parser Combinators with Macros, Béguet et al., Scala 2014][4].
Same principles as above, but uses macros as the metaprogramming framework of choice.

  [1]: http://research.microsoft.com/apps/pubs/default.aspx?id=65201
  [2]: http://infoscience.epfl.ch/record/203076?ln=en
  [3]: http://dl.acm.org/citation.cfm?id=734146
  [4]: http://infoscience.epfl.ch/record/200905?ln=en
