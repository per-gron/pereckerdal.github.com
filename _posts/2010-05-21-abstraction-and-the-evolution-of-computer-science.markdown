---
layout: post
title: Abstraction and the evolution of computer science
tags:
- language
- design
- thoughts
---

### The horizon of ideas

Ever since the first computer was constructed, the ways to explain to
them what computation they are supposed to do have become more and
more refined. The tools we use have become much better. Programming is
much easier now than it was twenty years ago.

Try writing a Facebook with the tools that were available 15 years
ago. I think it would be very hard to do. It would not be impossible,
but it would be so hard that no one would come up with the idea of
doing it. This becomes even more obvious when you look even further
back in time; Unix simply couldn't have been conceived by a programmer
who thinks in Assembler. It would be so hard to do that you wouldn't
even be able to imagine doing it.

Programming languages do, to some extent, define the problem space in
which programmers think.

As I see it, this is of key imporance to understand the role of the
evolution of programming languages. By designing languages that let
programmers think on a higher level you don't only allow them to work
faster, or write better, more reliable or more maintainable software:
A language that lets a programmer think on a higher level lets him
think about ideas that he wouldn't be able to grasp of in the lower
level language. *A higher level language widens the horizon of ideas
that are within reach.*

I believe that programming languages play a crucial role in the
evolution of computer science. This is why I think it is sad that
object-oriented programming has had such a prominent role in the last
decades, by many even thought of as the only "serious" paradigm. I see
OO as a local maximum: It is good, especially for some problem
domains, but it seems impossible to come much farther than where the
OO languages already are, because their ability to express abstraction
is so limited.

### Abstraction

In many areas, abstract is almost a negative word. It means that it is
up in the clouds there somewhere, not really tied to the real
world. Hey, we all know that the opposite of abstract is concrete. But
in mathematics and computer science, abstract has a completely
different connotation. In math and CS, the fact that an entity is
abstract doesn't mean that it is not concrete.

In computer science, abstraction is an exceptionally important
thing. Abstraction is what allows web programmers to think about
elements, events and stylesheets instead of register allocation,
memory copying, cache misses and different kinds of conditional jumps.

Just as in Paul Graham's
[Blub paradox](http://c2.com/cgi/wiki?BlubParadox), it is easy to look
on the layers of abstraction that we have built and be grateful for
them. Fewer people seem to understand (or care) that this implies that
there are higher levels of abstraction that we haven't yet invented
that will allow us to program things that are impractical or even
impossible (with our limited minds) to do now.

### Programming languages

These things make me believe that the future of computer science will
be highly influenced by what and how much we will be able to abstract
away from us programmer's poor and already over-encumbered minds. And
I think language design plays a crucial role in moving towards higher
abstraction.
