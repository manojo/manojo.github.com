---
layout: page
title: Home
tagline:
---
{% include JB/setup %}

Greetings!
==========

Greetings everyone! My name is Manohar Jonnalagedda. [You can call me
Mano](https://www.youtube.com/watch?v=iLkNPjbaPTk). I am a PhD student at the
[Programming Methods](http://lamp.epfl.ch) lab at EPFL, colloquially known as
the [Scala lab](http://www.scala-lang.org).

Research-wise, I am interested in embedded DSLs, their expressivity and their
performance. I have mostly worked on generative programming for parser
combinators and optimized data structures so far. This work uses the [LMS](http
://scala-lms.github.io) framework.

Otherwise, I enjoy music. I perform as an MC in the [Skankin' Society Sound
System](http://www.skankinsociety.ch/). You can find out more about sound
systems [here](http://en.wikipedia.org/wiki/Sound_system_%28Jamaican%29). You
can also experience sound system culture by showing up at one of our shows.

If you happen to find this stuff interesting, please do let me know (by
email/github for now)! If you can't wait for the next post, subscribe to the rss
feed [here](rss.xml).

Projects
==========================

[Parsequery](https://github.com/manojo/parsequery)
--------------------------------------------------
Keywords — parsing, macros, multi-stage programming, partial evaluation

A macro-based Scala library that eliminates abstraction overhead of composing
parsers. The resulting parsers exhibit performance close to hand-optimized C
code.

### Results
  * 2-3x performance improvements over Fastparse, a popular parser combinator library in Scala.
  * Order of magnitude performance improvement over standard parser combinators.
  * Within 75% throughput when compared to Nginx HTTP parser, hand-optimized in C.
  * Up to 120% throughput when compared to JQ JSON parser.
  * [Publication](https://infoscience.epfl.ch/record/203076?ln=en) at OOPSLA 2014.
  * [Presentation](https://www.youtube.com/watch?v=Cc6QrgqsoVI) at Scala World 2016.

[Staged fold fusion](https://github.com/manojo/staged-fold-fusion)
------------------------------------------------------------------
Keywords — fusion, deforestation, multi-stage programming, partial evaluation

An implementation of various fusion optimizations on list-like data structures,
as a library: the optimizations do not depend on a specific compiler, so can be
easily ported to other languages.

### Results

  * Insights for implementing practical, optimizing libraries: basis work for related projects.
  * [Publication](https://infoscience.epfl.ch/record/209021?ln=en) at Scala 2015. 

[Packrat parsing in Scala](scala-language.1934581.n4.nabble.com/attachment/1956909/0/packrat_parsers.pdf)
---------------------------------------------------------------------------------------------------------
Keywords — parsing, memoization

An implementation of packrat parsing in Scala. These parsers can handle a larger
class of parsers than before, and handle left recursion. The implementation is
now part of the Scala standard [parser combinator library](http://tiny.cc/nynaky).

Posts
==========================

{% assign posts_collate = site.posts %}
<div>
{% include JB/posts_collate %}
</div>
