---
layout: post
title: Symple hygienic Lisp-1 macros
tags:
- scheme
- lisp
- macros
---

Lisp macro systems come in different flavors. In my opinion, none of them are particularly good. Don't get me wrong; I love Lisp and I love macros. It's just that I haven't seen a Lisp macro system that lives up to its promise.

There are Common Lisp macros, which actually work kind of well. But [Lisp-2](http://hornbeck.wordpress.com/2009/07/05/lisp-1-vs-lisp-2/) makes functional programming cumbersome, and the fact that you can have two different symbols with the same name is just odd.

Then there are all the Scheme macro systems. You have fully unhygienic macro systems, which really doesn't scale in a Lisp-1 environment; there are fully hygienic macro systems like `syntax-rules`, but it is a full-blown language on its own, [it has super weird corner cases](http://groups.google.com/group/comp.lang.scheme/msg/eb6cc6e11775b619), and besides, you really want to be able to break hygiene occasionally; there are low-level hygienic macro systems like syntactic closures or explicit renaming, but they introduce a new concept for what an identifier is whose definition is *a whole lot* more complex than a plain symbol; finally there is `syntax-case`, which essentially combines the problems of `syntax-rules` and the problems of low-level hygienic macro systems, plus the fact that it introduces a new type of binding that is separate from run-time bindings and macro bindings.

For a long time I have asked myself why it isn't possible to make a macro system that [allows for true hygiene](/2010/05/18/the-two-faces-of-the-hygiene-problem/) while still leaving a possibility to break it, allows for execution of arbitrary Lisp code, work well in a Lisp-1 *and* be as intuitive as Common Lisp macros.

[It turns out that doing that is in fact possible](http://www.p-cos.net/documents/hygiene.pdf). This post outlines a way to design such a system. We will begin with R5RS Scheme, but with all macro facilities removed, and then add the neccesary features.


### `define-macro`

First, let's introduce a basic unhygienic Common Lisp-style macro facility. Here's a basic example of its use:

{% highlight scm %}
(define-macro (multiple-times times expr)
  `(let loop ((i ,times))
     (if (not (zero? i))
         (begin
           ,expr
           (loop (- i 1))))))

(multiple-times 2 (println "Hello, world!")) ;; Prints twice
{% endhighlight %}


### `gensym`

We will also need `gensym`. [`gensym` does not in itself provide all protection that is required for full hygiene](http://community.schemewiki.org/?hygiene-versus-gensym). However, it does solve half of the problem, and this system needs `gensym`.

Here's a version of the `multiple-times` that uses `gensym`:

{% highlight scm %}
(define-macro (multiple-times times expr)
  (let ((i (gensym))
        (loop (gensym)))
    `(let ,loop ((,i ,times))
       (if (not (zero? ,i))
           (begin
             ,expr
             (,loop (- ,i 1)))))))
{% endhighlight %}

This version of `multiple-times` lets us write things that wouldn't work in the non-`gensym` version:

{% highlight scm %}
(let ((i "Hello, world!"))
  ;; Print "Hello, world!" twice
  (multiple-times 2 (println i)))
{% endhighlight %}


### Shadowing

As I mentioned earlier, `gensym` in itself is not sufficient. Consider this code snippet:

{% highlight scm %}
(let ((if list))
  ;; We want this to print "Hello, world!" twice,
  ;; but instead we get an infinite loop!
  (multiple-times 2 (println "Hello, world!")))
{% endhighlight %}

An infinite loop is clearly not what we want. The problem is that we have shadowed `if` so that it refers to a function instead of the built-in conditional special form.

We need some way to refer to all bindings in scope, regardless of whether they are shadowed or not. There is a method to do this that I call *aliasing*. With aliasing, every binding (including top-level bindings) is renamed. For instance,

{% highlight scm %}
(let ((a 'a))
  a
  (let ((a 'b))
    a))
{% endhighlight %}

is transformed into

{% highlight scm %}
;; Within this scope, all references to let are renamed to let~0
(let~0 ((a~1 'a))
  ;; Within this scope, all references to a are renamed to a~1
  a~1
  (let~0 ((a~2 'b))
    ;; Within this scope, all references to a are renamed to a~2
    a~2))
{% endhighlight %}

`a~1` and `a~2` are arbitrary choices; they could be any symbol, as long as they are unique. They could be gensyms, for instance.

With aliasing,

{% highlight scm %}
(let ((if list))
  ;; We want this to print "Hello, world!" twice,
  ;; but instead we get an infinite loop!
  (multiple-times 2 (println "Hello, world!")))
{% endhighlight %}

would macro-expand into

{% highlight scm %}
(let~0 ((if~1 list~0))
  ;; Within this scope, all references to if are renamed to if~1
  ;;
  ;; This is still an infinite loop!
  (let~0 loop~1 ((i~1 2))
     (if~1 (not~0 (zero?~0 i~1))
           (begin~0
             (println~0 "Hello, world!")
             (loop~1 (- i~1 1))))))
{% endhighlight %}

which still doesn't work the way we want.


### `alias`

The missing piece in the puzzle is to introduce a new special form `alias`. `alias` works like `quasiquote`, except that all symbols that are aliased are renamed to their unique names. For instance `(alias if)` could evaluate into `if~0`, and

{% highlight scm %}
(let ((outer-if (alias if)))
  (let ((if 'if))
    ;; This evaluates to something like '(if~0 if~1)
    (list outer-if (alias if))))
{% endhighlight %}

Armed with `alias`, we are now able to define a fully hygienic version of `multiple-times`:

{% highlight scm %}
(define-macro (multiple-times times expr)
  (let ((i (gensym))
        (loop (gensym)))
    `(,(alias let) ,loop ((,i ,times))
       (,(alias if) (,(alias not) (,(alias zero?) ,i))
                    (,(alias begin)
                      ,expr
                      (,loop (,(alias -) ,i 1)))))))
{% endhighlight %}

With this version,

{% highlight scm %}
(let ((if list))
  (multiple-times 2 (println "Hello, world!")))
{% endhighlight %}

no longer gets stuck in an infinite loop, because the macro expansion uses the name that uniquely refers to the `if` special form directly.


### Convenience

Using `alias` explicitly all the time, like in the last definition of `multiple-times`, gets cumbersome. But remember that `alias` works like `quasiquote`. We could have defined `multiple-times` like this:

{% highlight scm %}
(define-macro (multiple-times times expr)
  (let ((i (gensym))
        (loop (gensym)))
    (alias
     (let ,loop ((,i ,times))
        (if (not (zero? ,i))
            (begin
               ,expr
               (,loop (- ,i 1))))))))
{% endhighlight %}

Neat, no?


### How is this different from Clojure?

This system is very similar to Clojure's macros. There are a few key differences:

<ol>
<li><p>Clojure hijacks <code>quasiquote</code> so that you can't use it for creating plain lists, because symbols are renamed. Yuck.</p></li>
<li>
<p>Clojure's hygiene works on a per-namespace basis, and breaks down when you do complex things with scopes. For example, you can't directly translate this into Clojure:</p>

{% highlight scm %}
(let ((a 'a))
  (define-macro (mac name)
    (alias
     (list a ,name)))
  (let ((a 'b))
    ;; Evaluates to '(a b)
    (mac a)))
{% endhighlight %}

<p>Well, why does this matter, you might ask. In my opinion programming language features should be robust, in the sense that they work well even when you abuse them. That makes it possible for programmers to do things that the language designer didn't anticipate, which is important. The brilliant people working on <a href="http://racket-lang.org">Racket</a> have abused hygienic macros like crazy, and many of the techniques they use wouldn't work if hygiene doesn't work at the local scope level.</p>
</li>
</ol>


### Why this is great

The macro system I have described in this post is easy to use, allows execution of arbitrary Lisp code, allows the user to choose whether to break hygiene or not, is lightweight, has semantics that is very easy to reason about and does not introduce a new non-symbol identifier concept to the language. Furthermore, it is very simple to implement and no special algorithmic tricks are required to implement a macro expander that takes linear time with respect to code size.

Furthermore, it is straightforward to design a module system that works with this macro system.
