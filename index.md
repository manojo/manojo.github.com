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

Posts
==========================

{% assign posts_collate = site.posts %}
<div>
{% include JB/posts_collate %}
</div>
