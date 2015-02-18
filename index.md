---
layout: page
title: Home
tagline:
---
{% include JB/setup %}

Greetings!
==========

Greetings everyone! My name is Manohar Jonnalagedda. [You can call me Mano](https://www.youtube.com/watch?v=iLkNPjbaPTk).
I am a PhD student at the [Programming Methods](http://lamp.epfl.ch) lab at EPFL, colloquially known
as the [Scala lab](http://www.scala-lang.org).

Research-wise, I am interested in embedded DSLs, their expressivity and their performance. I have mostly worked
on generative programming for parser combinators and optimized data structures so far. This work uses the
[LMS](http://scala-lms.github.io) framework.

Otherwise, I enjoy music. I perform as an MC in the [Skankin' Society Sound System](http://www.skankinsociety.ch/).
You can find out more about sound systems [here](http://en.wikipedia.org/wiki/Sound_system_%28Jamaican%29).
You can also experience sound system culture by showing up at one of our shows.

About this site
===============

The reason for this site is that it was about time to get one. It's good to have a place where people
can check your 'professional' self out. Also, I have the undesirable habit of forgetting something
that I learnt a while ago, only to have to re-read and re-understand it. So in the blog here, I
will mostly write about stuff that I think I understood, so in case I forget, I can come back and see
how I got it in the first place. Maybe it will improve my writing skills too!

Most recent posts (last 3)
==========================

<div>
  <ul>
  {% for post in site.posts limit:3 %}
    {% if post.title %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endif %}
  {% endfor %}
  </ul>
</div>
