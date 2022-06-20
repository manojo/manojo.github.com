---
layout: page
title: Home
tagline:
---
{% include JB/setup %}

Greetings!
==========

![]({{ site.url }}/images/manohar.jpeg){:height="200px" width="200px"}.

Hi! My name is Manohar Jonnalagedda. [You can call me
Mano](https://www.youtube.com/watch?v=iLkNPjbaPTk). I like designing tools that
substantially improve the quality of life of their users, and I love working in
teams to build these. By doing that, I have gained expertise in
privacy-enhancing technologies, language design, formal methods, and compilers.
I have experience in building software from scratch, as well as contributing to
larger projects. I occasionally write and talk about them.

I am currently involved in designing the future of privacy-preserving
programming at Inpher. In the past I worked combining program synthesis and
machine learning for a best-of-both worlds approach, at the wonderful [MSR
India](https://www.microsoft.com/en-us/research/lab/microsoft-research-india/).
I learnt all about compilers and DSLs during my PhD at the [Programming
Methods](http://lamp.epfl.ch) lab at EPFL, colloquially known as the [Scala
lab](http://www.scala-lang.org).

Otherwise, I enjoy music. I perform as an MC in the [Skankin' Society Sound
System](http://www.skankinsociety.ch/). You can find out more about sound
systems [here](http://en.wikipedia.org/wiki/Sound_system_%28Jamaican%29). You
can also experience sound system culture by showing up at one of our shows.

If you happen to find this stuff interesting, please do let me know (by email/github)!

Projects
========

Speakeasy
---------

Speakeasy is a library for composing privacy-enhancing technologies. It allows
you to express the general flow of secret and non-secret data as a logical
graph, and then offers a way to analyze this graph and execute it on different
backends depending on the need. As a result, a user not only gets
domain-specific annotations and error-messages for the privacy-preserving part
of their code, they can also run full pipelines of non-trivial programs defined
in a single, cohesive environment.

[[video](https://www.youtube.com/watch?v=HI8QnC8NnI4) | [slides](https://jakob.odersky.com/talks/2021-scalacon.pdf)]

An earlier [talk](https://www.youtube.com/watch?v=44EL11N3tOs) details how we
initially designed an external DSL for multi-parti computation. Slides can be
found [here](https://jakob.odersky.com/talks/2019-scaladays.pdf).

Synthesis and machine learning for heterogeneous extraction
-----------------------------------------------------------

In this project, we combine machine learning and program synthesis in order to
improve automated data extraction from semi-structured documents. The idea is to
let machine-learning techniques identify common patterns (e.g. flight number,
departure time) in a small training dataset, and use program synthesis to
generate a program based on these examples to extract from a test dataset. This
hybrid approach helps not only extract patterns better, it also helps a user
understand (via the generated program) how some of these patterns have been
identified.
[[publication](https://www.microsoft.com/en-us/research/uploads/prod/2019/04/HeterogeneousExtraction.pdf)]

[Parsequery](https://github.com/manojo/parsequery)
--------------------------------------------------
Keywords — parsing, macros, multi-stage programming, partial evaluation

A macro-based Scala library that eliminates abstraction overhead of composing
parsers. The resulting parsers exhibit performance close to hand-optimized C
code.

##### Results
  * 2-3x performance improvements over Fastparse, a popular parser combinator library in Scala.
  * Order of magnitude performance improvement over standard parser combinators.
  * Within 75% throughput when compared to Nginx HTTP parser, hand-optimized in C.
  * Up to 120% throughput when compared to JQ JSON parser.
  * [Publication](https://infoscience.epfl.ch/record/203076?ln=en) at OOPSLA 2014.
  * [Presentation](https://www.youtube.com/watch?v=Cc6QrgqsoVI) at Scala World 2016.
  * [Dissertation](https://infoscience.epfl.ch/record/222871?ln=en) related to the topic.

[Staged fold fusion](https://github.com/manojo/staged-fold-fusion)
------------------------------------------------------------------
Keywords — fusion, deforestation, multi-stage programming, partial evaluation

An implementation of various fusion optimizations on list-like data structures,
as a library: the optimizations do not depend on a specific compiler, so can be
easily ported to other languages.

##### Results

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
