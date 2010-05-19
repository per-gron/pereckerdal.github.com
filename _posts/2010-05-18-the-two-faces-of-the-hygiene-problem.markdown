---
layout: post
title: The two faces of the hygiene problem
tags:
- scheme
- lisp
- macros
---

This post is an introduction to the problem that hygiene systems for
Scheme try to solve. When reasoning about this, I have often found it
useful to divide these problems into two categories, which I will also
describe here.

As most Schemers will know, writing macros isn't as painless as we
wish it could have been. One of these issues is hygiene; we want
macros to be hygienic. But what does that mean and why do we want it?

In this article I'll be writing code in Gambit-C Scheme.

Suppose we want to write a macro `ten-times` that runs code ten
times. We could write it like this:

{% highlight scm %}
(define-macro (ten-times . x)
  `(let loop ((i 10))
     ,@x
     (if (> i 1)
         (loop (- i 1))))) 
{% endhighlight %}

And that would work most of the time, for instance
with `(ten-times (println "Hello"))`. But writing

{% highlight scm %}
(let ((i 3))
  (ten-times
   (set! i (+ 1 i))
   (println "Number: " i))) 
{% endhighlight %}

results in an infinite loop! This is not good; it breaks
abstraction. However, this problem is easy to solve with gensyms; we
can define the macro as

{% highlight scm %}
(define-macro (ten-times . x)
  (let ((loop-gs (gensym))
        (i-gs (gensym)))
    `(let ,loop-gs ((,i-gs 10))
        ,@x
        (if (> ,i-gs 1)
            (,loop-gs (- ,i-gs 1)))))) 
{% endhighlight %}

The basic idea is that, for each binding that the macro introduces, it
has to use a generated symbol that is guaranteedly unique.

Some Schemers loathe this "solution". But Common Lispers have done
this for ages, and this solution works for them. In my opinion, if
this was the only problem with hygiene, there would be no need to
implement hygienic macro systems. There is, however, another problem
with `ten-times` above. This code also results in an infinite loop:

{% highlight scm %}
(let ((- +))
  (ten-times (println "Hello")))
{% endhighlight %}

Gensyms don't help here. The problem with this is that the code that
the macro expands to must be able to use library functions and special
forms (in this case `let`, `-` and `if`) to be able to do something
useful. But because the code is interpreted in the lexical environment
of the expansion, this can lead to problems.

Hygienic macro systems for Scheme usually solve both of these problems
by making sure that the code that the macro expands to are evaluated
in the lexical environment where the macro was defined. They also
often add special means to break hygiene when the macro writer wants that.
 
The second might seem unlikely to ever occur; why would anyone want to
override the meaning of `+`? As I see it, there is primarily one
important case where this happens, and that's when you combine modules
and macros. Let's say you have a module *A* that imports module *B*
and exports a macro `mac`. `mac` uses some functions from *B*. If you
don't have hygiene and want to be able to use `mac`, you'd have to
import both *A* and *B*. This is bad, because you don't want to have
to expose to the users of `mac` how it's implemented. And you lose one
important thing that you can do when you combine hygiene, macros and
modules: DSLs. PLT Scheme uses this to great effect. If you have all
those three features, it is possible to write code in different
languages (as long as they macroexpand into Scheme) and mix code
freely. This is a really cool feature.
