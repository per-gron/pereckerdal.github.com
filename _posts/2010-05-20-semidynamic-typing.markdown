---
layout: post
title: Semi-dynamic typing
---

Different languages have different typing diciplines. There are
languages with strong static typing, like Haskell. There are languages
with weak dynamic typing, like Javascript. There are languages with
static but weak type systems, like C. There are languages with dynamic
but strong type systems, like Scheme.

With weak typing, I mean types that don't map very robustly to the
actual values in the language. Such systems often have lots of
implicit type coercions and/or type casts. In my experience this
almost invariably leads to needlessly fragile programs. "Which values
were it that count as false again?" is a question i often ask myself
when writing Javascript, and bugs that are because I forgot that the
empty string counts as false and similar is not uncommon.

I seriously don't get such designs. The fact that I don't have to
write `str != ''` doesn't make my programs much shorter, easier to
write or easier to read.

So, as you might have understood by now, I'm a big fan of strong
typing, regardless of whether it's static or dynamic. I like to know
that when I have a string, it is a string, nothing more, nothing
less. This blog post is going to be about strong static typing and
strong dynamic typing. (Although it will, at least to some extent, be
applicable on weak typing as well.)

### Why does it have to be either static or dynamic typing?

Dynamic typing has many advantages. It's possible to write code
quickly and effortlessly, and you can focus entirely on the problem at
hand, without having to worry about fighting the type system of the
language. In most cases, unit tests provide sufficient assurance that
the programs that you write are correct. Dynamically typed languages
are often very suited to exploratory programming, prototyping and
quick and dirty hacks.

Static typing also has many advantages. If the chosen language has a
good type system, it will help you to write correct programs by
pointing out many of the errors that would otherwise only have
appeared at run-time. Unit tests become less necessary. If you want to
write robust software, static typing is a great asset.

There is a great ongoing debate about static versus dynamic typing. I
think most people realise, deeply inside, that the answer to the question
depends on the problem at hand. But it's always fun to flame on the
Internetz, so the war continues.

Here's one thing that I have been thinking of lately: One thing that
has struck me when I have studied different programming languages'
type diciplines is that virtually every language is either static or
dynamic with regards to typing. Why is there no middle ground?

For a long time I have tried to figure out a way to define that middle
ground that I had a feeling existed, without success. But recently the
puzzle seems to begin to make a picture that makes sense.

### Types in Scheme

The last years, I have been coding a lot of Scheme. There are many
things that I love about that language, but there are also some things
that annoy me. One of those things is that it's typing feels
needlessly dynamic, or rather, the language is defined in such a way
that the compiler can't really help me at all in finding even the most
obvious errors. I mean, it ought to be able to find at least some
wrong-number-of-arguments errors and the most glaring type mismatches,
like `cdr`ing a string.

These super-obvious errors are easy to spot and remove when you write
code, but when you change code things often break like that in places
that you forgot were related. I feel that the compiler should be able
to help out.

Also, making the compiler more aware of types should help with other
things as well, like optimizing and other kinds of semantic analysis.

### Semi-dynamic typing

What I hope to be able to do is a language which by default is
entirely dynamic, but allows adding type annotations, and, to the
extent that the compiler is able to analyze the program, gives
compile-time errors where appropriate.

In a pure dynamic dicipline, you can say that the dynamic language
doesn't allow type errors at compile-time; the compiler only checks
that the input program is syntactically correct so that it is able to
infer its meaning unambigiously. In a pure static dicipline, you can
say that the compiler will check every expression's type and see if
they are correct. If the compiler cannot guarantee their correctness,
the program will be rejected.

I think it's possible to find a middle ground by saying that the
compiler is allowed to compile any syntactically valid program, but if
it finds an expression that it can prove that, when run, will always
produce a type error, it is allowed to throw a compile-time
error. This semi-dynamic dicipline doesn't have all the advantages of
dynamic typing, and does not have all the advantages of static typing,
but I think it's interesting nontheless.

Adding this to Scheme can be as simple as adding one built-in
procedure of the language. In this blog post I'll call it
`assert`. Assert is a procedure that takes one argument, and if that
argument is `#f`, an error is thrown. Furthermore, if the compiler
finds an assert expression that it can determine statically to always
fail, it is allowed to not compile it. For instance,

{% highlight scm %}
(assert #t) ; => Returns #!void or something to that effect
(assert #f) ; => Error, either compile-time or run-time

(define (car pair)
  ;; A safe definition of car, based on an unsafe version called
  ;; unsafe-car.
  (assert (pair? pair))
  (unsafe-car pair))
{% endhighlight %}

With the definition of `car` above, it should be relatively
straightforward for a compiler to see that `pair?` is a type
predicate, and then understand that `car` is a function that takes a
pair as its parameter. It's obviously also possible to add syntactic
sugar for this, for instance

{% highlight scm %}
(define (car pair:pair?)
  (unsafe-car pair))
{% endhighlight %}

I haven't tried what it feels like to program in this style, or if
it's possible to write a
[sufficiently smart compiler](https://c2.com/cgi-bin/wiki?SufficientlySmartCompiler)
(oh we all love that term, don't we?), but I think it might be nice to
code that way, and I think it might be possible to implement a
compiler that understands this.

One interesting property of `assert` is that it is dead easy to add it
to an existing Scheme implementation; a completely valid (although not
the most useful) definition of `assert` is

{% highlight scm %}
(define (assert val)
  (if (not val)
      (error "Assertion failure")))
{% endhighlight %}

Another thing with this way of doing typing is that it is easy to
define, but it allows a myriad of ways to implement it, and is open to
a lot of experimentation with regards to exactly how smart the
compiler should be. I believe that this is a good trait for a language
specification that hopes to last the test of time. But it also means
that the set of programs that a compiler accepts might be different to
the set of programs that another compiler for the same language
accepts. That might be a problem, but I don't think so, because both
compilers must accept every valid program.

### The consequences of this style

As I said, I haven't coded in this style yet, but I think that the
effect of this is that it will be possible to jot down quick
prototypes easily, and as the programs mature, you can add more and
more type checks, resulting in an increasingly stable program. If that
is how it works in practise, this is in my opinion a very cool
language feature.

If type annotations are added for most top-level procedures, which is
good style anyways, if only for documentation purposes, I think the
compiler will be able to statically find most of the errors that we
expect to get caught by a semi-strong statically typed language like
for instance Java.

### Update

I found out about the concept of
[soft typing](https://c2.com/cgi-bin/wiki?SoftTyping), which is
closely related to what I describe here. Other interesting links are
[MrSpidey](http://www.plt-scheme.org/software/mrspidey/),
[MrFlow](http://www.plt-scheme.org/software/mrflow/), papers about
set-based analysis
[here](http://www.cs.rice.edu/CS/PLT/Publications/Scheme/), and
possibly the references
[here](http://download.plt-scheme.org/doc/103p1/html/mrspidey/node26.htm).
