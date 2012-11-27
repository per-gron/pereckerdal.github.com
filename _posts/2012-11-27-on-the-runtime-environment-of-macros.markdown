---
layout: post
title: On the runtime environment of macros
tags:
- scheme
- lisp
- macros
---

A problem that frequently cripples Lisp macro systems that allow execution of arbitrary code is that it is unclear what functions that are available to macros. Almost all systems allow things like this:

{% highlight scm %}
(define-macro (mac . numbers)
  `(list
    ,@(map (lambda (x) (+ x 1))
           numbers)))

(mac 1 2 3 4) ;; Evaluates to '(2 3 4 5)
{% endhighlight %}

However, few systems allow you to do

{% highlight scm %}
(define (incr x) (+ x 1))

(define-macro (mac . numbers)
  `(list
    ,@(map incr
           numbers)))

(mac 1 2 3 4) ;; Error; incr is not defined in the macro's scope
{% endhighlight %}

The reason this doesn't work is that macro expansion is done before compilation of `incr`. It doesn't make sense for `mac` to refer to `incr`. This is a serious problem when writing non-trivial macros; you really need some way to access helper functions from macros.

Many Lisps "solve" this by allowing you to `eval` within macros. For instance, you could write something like

{% highlight scm %}
(define-macro (define-incr)
  (eval
   `(define (incr x) (+ x 1))))
(define-incr)

(define-macro (mac . numbers)
  `(list
    ,@(map incr
           numbers)))

(mac 1 2 3 4) ;; Evaluates to '(2 3 4 5)
{% endhighlight %}

This is not only ugly and messy, it also becomes ridiculously complicated to maintain with large code bases.

Some more sophisticated Lisp implementations solve this problem through their module system:

{% highlight scm %}
(import-for-macros module-that-has-incr)

(define-macro (mac . numbers)
  `(list
    ,@(map incr
           numbers)))

(mac 1 2 3 4) ;; Evaluates to '(2 3 4 5)
{% endhighlight %}

This is a whole lot better, because it doesn't break down on large code bases. Some Scheme implementations, for instance [Chez](http://www.scheme.com) and <a href="http://en.wikipedia.org/wiki/Ikarus_(Scheme_implementation)">Ikarus</a> even [infer on which execution levels identifiers are used](http://www.cs.indiana.edu/~dyb/pubs/implicit-phasing.pdf) so that you can just write `import` without having to worry about whether you use the module for macros or on runtime.


### The `library` special form

While this model works, it's rather inconvenient to create a new file with utilities if you just need one that is used in one file.

When experimenting with module and macro systems, I've come up with a nice design of a special form that solves this problem elegantly. This might already exist, but I have not seen it anywhere else, so I decided to describe it here.

The basic idea is to create a special form that defines an anonymous module. For lack of a better name, I call it `library`. Here's an example of `library` in action:

{% highlight scm %}
(library
 (define (incr x) (+ x 1)))

(define-macro (mac . numbers)
  `(list
    ,@(map incr
           numbers)))

(mac 1 2 3 4) ;; Evaluates to '(2 3 4 5)
{% endhighlight %}

`library` is similar to Java's anonymous inner classes. It behaves as if you would have moved the contents of the `library` form into a separate file and imported it. Just like it makes sense to allow `import` within local scopes, it also makes sense to allow `library` there.

An important aspect of `library` is that it clears the scope: You can't do things like

{% highlight scm %}
(define (incr x) (+ x 1))

(library
 (define (incr2 x) (incr (incr x))))
{% endhighlight %}

because within the `library` form, `incr` is not defined. It is possible to do it the other way though:

{% highlight scm %}
(library
 (define (incr x) (+ x 1)))

(define (incr2 x) (incr (incr x)))
{% endhighlight %}

I like `library` because it is easy to use, easy to reason about, it makes certain things easier to write and in my experience it is rather easy to implement. In fact, implementing `library` is great for a module system even if you don't expose the form in a public API because it results in neat module system implementation code.
