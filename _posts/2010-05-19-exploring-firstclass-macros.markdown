---
layout: post
title: Exploring first-class macros
tags:
- scheme
- lisp
- macros
---

I know that first-class macros in many circles are regarded as being
highly blasphemous. The arguments against them are often loud and
desperate. In this post I'm going to try to ignore these sentiments
(yes, I definitely have them too) and try to explore what consequences
first-class macros actually have.

Maybe I should define exactly what I mean with first-class macros, but
I won't bother doing that now, I think most people will understand
what I mean well enough if I just say that I want to be able to use
macros as any other function. I should also say that the thoughts I
present here aren't mature at all: This post probably contains lots of
glaring errors and inconsistencies. But it's time to begin!

### Macros that ought to be procedures

The most common example that I see is about macros that people want to
be able to use as a procedure, `and` is an example. In R5RS, it is
impossible to do

{% highlight scm %}
(define (fold fn accum list)
  (if (null? list)
      accum
      (fold fn
            (fn accum
                (car list))
            (cdr list))))

(fold and #t '(#t #f #t #t)) ; => should return #f
{% endhighlight %}

If R5RS had first-class macros, this would work as expected. (In fact,
this code works in [Heist](http://github.com/jcoglan/heist), a Scheme
interpreter written in Ruby.) Personally, I don't think this is a very
strong argument for first-class macros; even though it would have
given the correct answer, I don't know of any way to compile this into
efficient code. Even though it shouldn't be the only thing that drives
the design of a language, efficiency certainly is an important
factor. I believe that the sweet spot in language design is to be nice
both to the programmer and to the computer. And the problem above
could easily be implemented with a non-short-circuiting `and` or, even
more elegantly, with lazy evaluation.

But macros can be used for more than just procedures that would
benefit from other evaluation strategies; I often hear Haskellers
complain about this, but I never hear them complaining about that they
can't pass `let` as an argument to a function. That doesn't make
sense, because `let` is syntactic sugar for `lambda`, and lazy
evaluation doesn't solve that.

### Effects on the concept of phase separation

One of the big contributions that the PLT Scheme people have made to
the Scheme community is to figure out a way to reason about how to
make a clean distinction between macro expansion time and run
time. The main paper on this topic is called [Composable and
Compilable Macros – You want it
*when*?](http://www.cs.utah.edu/plt/publications/macromod.pdf). It is
well worth reading.

The main idea is that, to make macro expansion work in practise and
not be hopelessly fragile, macros must not be able to access run-time
variables, and run-time code must not be able to access macroexpansion
time variables. This is done by introducing the concept of a
*syntactic tower*, a metaphorical tower of Scheme virtual machines,
where the first level is the run-time level, and the only way to
communicate between the levels is that any level can invoke macros at
the next level, and those macros return s-expressions to the level
below.

With first-class macros, separating the macro-expansion phase from the
run-time phase is nonsensical; macros are expanded for every
invocation, completely interleaved with the rest of the execution of
the program. (Note that if macros are used as if they were not first
class, it is often possible to expand them at compile time, as it
makes no observable difference to do it once per invocation or just
when the code is compiled)

### Effects on what s-expressions can contain

In R5RS, s-expressions are defined basically to contain any object
that has an external representation.

If you closely look at how R6RS syntax-case macros are defined, you
will see that it is impossible to make macros that embed complex
values in the resulting s-expression. In particular, it is impossible
to write a macro that leaves a closure inside of the returned
s-expression. This is because doing so would violate phase separation;
if macros could leave closures in their output, it would be possible
for the code in the phase below to access the macro phase (and vice
versa).

Because first-class macros remove the need of separating expansion
phases, this restriction on the contents of an s-expression can be
lifted; you can, without losing consistency, define s-expressions to
be any object in the language.

### Effects on hygiene

Now it will finally begin to become interesting. Here I will show that
it is possible to exploit a combination of the fact that macros are
first class and that s-expressions can contain closures to take a
different approach to hygiene. Before I begin, I should give credit to
Matt Might, who introduced this concept to me in
[this fine blog post](http://matt.might.net/articles/metacircular-evaluation-and-first-class-run-time-macros/).

In a previous blog post, i have talked about that you can divide the
problem of writing hygienic macros into two separate issues; one that
can be solved by proper use of gensyms, and one that can't. Here, I
will focus only on the problem that can't be solved with gensyms. What
we're trying to do is to write macros whose expansions refer to macros
and procedures that are guaranteed to be looked up in the lexical
scope of the macro and not the scope of the macro expansion.

With a Scheme with first-class macros and s-expressions that can
contain any object, including macros and procedures, it becomes easy
to write hygienic macros: (I assume that this is in a language where
procedures and macros self-evaluate, because this becomes more
natrual, but it would work otherwise as well.)

Continuing with the example from the previous blog post:

{% highlight scm %}
(define-macro (ten-times . x)
  (let ((loop-gs (gensym))
        (i-gs (gensym)))
    `(let ,loop-gs ((,i-gs 10))
        ,@x
        (if (> ,i-gs 1)
            (,loop-gs (- ,i-gs 1)))))) 
{% endhighlight %}

If `ten-times` is modified to be like this, it is fully hygienic:

{% highlight scm %}
(define-macro (ten-times . x)
  (let ((loop-gs (gensym))
        (i-gs (gensym)))
    `(,let ,loop-gs ((,i-gs 10))
        ,@x
        (,if (,> ,i-gs 1)
             (,loop-gs (,- ,i-gs 1)))))) 
{% endhighlight %}

It works because the actual macros and procedures are embedded into
the s-expression, not the names of them, so there is no room for
inadvertent variable capture because there are no names.

This truy excites me. So simple, so clean. It works without having to
introduce a horde of new complex concepts like identifiers, special
syntax s-expressions.. the list could go on. Let me quote a sentence
that has been in several RnRS specifications:

> Programming languages should be designed not by piling feature on
> top of feature, but by removing the weaknesses and restrictions that
> make additional features appear necessary.

To me, this is exactly what this is about. And we have just begun.

### Module systems

Another issue that the Scheme community has fought with for a long
time is how to modularise development. PLT Scheme, having as goal to
make Scheme viable for "real-world" development has attacked this
issue and made an impressive module system (later adopted by R6RS)
that integrates beautifully with the rest of the language and adds a
whole lot of possibilities. But, just like syntax-case, it solves the
problem by adding more stuff to the language. If you want to design a
language that works for real development *now*, that is a viable
trade-off, but it's definitely not optimal, and IMO not in the spirit
as defined in the introduction of the latest RnRS specifications.

The main problem I see with R6RS modules is that there are so many
different but incompatible ways to design a system in the veins of
R6RS. It contains **lots** of design desicions that have been made
only because they had to be made. In my opinion, a beautiful language
feature has a feeling to it that the way it is done is more or less
the only way to do it; compare R6RS modules with `lambda`, `cons` or
`call/cc`. Practical programming will always contain this kind of
trade-offs, but designs that should last must have a more time-less
feel to them than R6RS modules have.

Enough ranting, let me explain what I'm thinking of. I don't have the
space (or time) to give a full implementation of this, but I'll sketch
it:

First, we define what a module object is. For this demonstration, we
can define it to be an a-list from symbols to values, where each entry
represents an exported name. With that, we can define a `with-module`
macro, that takes a module as an argument and body code, and expands
into a `let` form which binds the exported names of the module. For
instance,

{% highlight scm %}
(with-module `((a . ,(lambda () 'a))
               (b . ,(lambda () 'b)))
  (cons (a) (b)))
{% endhighlight %}

could expand into something like

{% highlight scm %}
(let ((a #<procedure a>)
      (b #<procedure b>))
  (cons (a) (b)))
{% endhighlight %}

(The `with-module` macro must be implemented as a macro that expands
into a local macro definition to make it possible for the first
argument of `with-module` to be evaluated and not just be an
s-expression, but I won't go into further detail in how that is done.)

There are many ways to implement macros that help in the creation of
module objects, but they are not very difficult to make. The point is
that this is a module system that needs no extra support from the
language, and it is possible to play around and define different
module systems. Also, there is nothing that hinders modules from using
modules from other module systems, they just have to make sure to use
the right way to import them.

I find this remarkable. First class macros have issues, and it is not
very easy to implement a compiler that compiles this into efficient
code. But I don't think it's impossible. And the arguments I have
outlined above do at least justify looking into it. It fits so well
into the spirit of the introduction to the R5RS specification.

### Some other interesting side-effects 

With this, it becomes interesting to introduce a special form
that takes an argument that it evaluates in an empty lexical
environment. That becomes useful for instance when implementing module
systems.

Some other interesting side-effects of having first-class macros is
that it becomes possible to implement `eval` as a macro. It is also
possible to implement `apply` as a macro. (It becomes impossible to
implement it as a procedure, because its first argument might be a
macro.) This is interesting, and is also indirectly what many people
criticise when argumenting against first class macros.

There are obviously also problems with first class macros, the main
objection I have is that they potentially break abstraction; you can
pass a macro to a procedure that expects a procedure and hijack its
internals. I'm thinking about what complications that has and what one
can do to hinder the problems that arise.
