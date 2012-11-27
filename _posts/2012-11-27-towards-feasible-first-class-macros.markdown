---
layout: post
title: Towards feasible first-class macros
tags:
- scheme
- lisp
- macros
---

I have previously written about [some of the cool things that you can do with first-class macros](/2010/05/19/exploring-firstclass-macros/). It is easy to create an interpreter with support for first-class macros, but it is really really hard to compile code with first class macros. In fact, I have never seen anyone succeed in doing it. There is also a deep semantic problem with first-class macros as defined in the previous link.

This post is about a technique that solves the semantic problem and also makes it easier to implement a true compiler for a language with first-class macros.

### The semantic problem

First, let me describe the semantic problem with first-class macros: They break abstraction.

A really important property of languages like Scheme is that if you define a function with a lambda, you can reason about it much like a mathematical function. In particular, as long as it does the same thing at all times, you can change its implementation however you wish.

Consider the following function to reverse a list:

{% highlight scm %}
(define (reverse l)
  (let loop ((lst l) (accum '()))
    (if (null? lst)
        accum
        (loop (cdr lst)
              (cons (car lst)
                    accum)))))
{% endhighlight %}

This is a correct definition, but it can be refactored into smaller reusable pieces:

{% highlight scm %}
(define (fold f init l)
  (let loop ((lst l) (accum init))
    (if (null? lst)
        accum
        (loop (cdr lst)
              (f (car lst)
                 accum)))))

(define (reverse l)
  (fold cons '() l))

;; Use fold to implement a function that sums the elements in a list
(define (sum l)
  (fold + 0 l))
{% endhighlight %}

This is nice. We have refactored the code in a way that makes it more reusable, and at the same time it's easy to prove that the new definition of `reverse` is equivalent. This type of reasoning and refactoring is what functional programming really excels at.

Let's see what happens when we throw first-class macros into the mix.

{% highlight scm %}
(define (fold f init l)
  (let loop ((lst l) (accum init))
    (if (null? lst)
        accum
        (loop (cdr lst)
              (f (car lst)
                 accum)))))

(define-macro (hijack val accum)
  'loop)

(fold hijack 0 '(1 2 3)) ;; Returns the value of loop
{% endhighlight %}

This was not what we intended when defining `fold`! We never wanted code outside of `fold` to be able to access `loop`. This clearly breaks abstraction. What's worse, we can no longer change the definition of `fold` without potentially breaking code that uses it. In this case, we can no longer rename or remove the `loop` binding. In general, we can't change *anything* in a higher order function without potentially breaking places where it's used.

This also highlights why it's so hard to compile code with first-class macros. As soon as higher order functions are involved, all bets are off. It's easy to construct an example where `fold` would have to be recompiled on every iteration.


### A solution

When I was thinking of this problem I asked myself if I even want it to be possible to pass a macro to `fold`. It doesn't make sense to do that; the `f` parameter to `fold` should take the two values and return a value, it's agains the definition of `fold` to do anything else. So I thought, maybe I could simply write

{% highlight scm %}
(define (fold f init l)
  (assert (procedure? f))
  (let loop ((lst l) (accum init))
    (if (null? lst)
        accum
        (loop (cdr lst)
              (f (car lst)
                 accum)))))
{% endhighlight %}

This way, `fold` is no longer a leaky abstraction in the sense I described above.

Continuing this trail of thought, I realized that disallowing macros to be sent as parameters into higher order functions is probably even a sane default. To do this, you could change the language so that there are two types of bindings: normal bindings and bindings that can refer to macros. With this, I don't mean that normal bindings can't refer to a macro, only that when a normal binding is invoked, it must be a function. For instance:

{% highlight scm %}
(define fun (lambda ()  'a-function))
(define mac (macro  () ''a-macro))

(fun) ;; Evaluates to 'a-function
(mac) ;; Error: invoking a macro as a procedure

(define-macro fun2 fun)
(define-macro mac2 mac)

(fun2) ;; Evaluates to 'a-function
(mac2) ;; Evaluates to 'a-macro
{% endhighlight %}

In order to write a `fold` that allows `f` to be a macro, you'd have to declare that explicitly:

{% highlight scm %}
(define (fold f init l)
  (let-macro ((f-mac f))
    (let loop ((lst l) (accum init))
      (if (null? lst)
          accum
          (loop (cdr lst)
                (f-mac (car lst)
                       accum))))))
{% endhighlight %}

With this restriction, you can write higher order functions as usual, without having to worry about weird abstraction leakage. At the same time, macros are still first class objects and most of the cool things you can do with them remain possible.

Furthermore, this massively simplifies compiler implementation: The cases where what looks like a function call might actually be a macro invocation are much less common, are syntactically obvious, and will in fact likely actually be macro invocations. A compiler will be able to expand a macro invocation at compile time when it's able to prove that the macro invocation always refers to the same macro, and that the invocation is referentially transparent.
