---
layout: post
title: Three programming language design principles
tags:
- language
- design
- lisp
---

## Timelessness and smallness

Computer Science is a very young field. What should we be working towards?

Math is timeless. Why can't programming be? I hope that programming
will evolve into that.



## CSS rant



### Difficult problems

"Big problems are terrifying. There's an almost physical pain in
facing them. It's like having a vacuum cleaner hooked up to your
imagination. All your initial ideas get sucked out immediately, and
you don't have any more, and yet the vacuum cleaner is still
sucking."

http://paulgraham.com/procrastination.html

Svart mur


### Success typing...

Saker som språket vill försöka analysera statiskt:
* Huruvida datastrukturer är konstanta

* Huruvida funktioner är rena (konservativ analys), dvs
  * Givet ekvivalenta parametrar ska den alltid returnera samma värde
    och bete sig på samma sätt
  * Inte sluter över icke-konstanta värden
  * Inte anropar icke-rena funktioner
  * Och därmed också vilka namn som en funktion sluter över

* Ett försök till att makroexpandera kod när den kompileras:
  * Makron som anropas som är rena funktioner kan makroexpanderas

* Vad som krävs för att kunna göra en analys av ett uttryck:
  * Den lexiska miljön. Alltså måste all kod ovanför vara makroexpanderad.

CPS-transformation?

abstrahera och försöka karakterisera indata och utdata för olika
algoritmer. man kanske kan hitta en algoritm som kombinerar ihop det
på något sätt.

Transformationer:
* S-exp flödesanalys för att makroexpandera uppenbara makron
* Typslutledningsalgoritm
* CPS-transformation
* Renhetsanalys av datastrukturer
* Renhetsanalys av funktioner

Evalueringsalgoritmer:
* Small-step interpreter, takes s-expressions with macros as input
* Compiler, takes CPS (without macros) as input

Saker som funktioner ska annoteras med:
* Renhet (någon av ja, inte säkert, inte beräknat än)
* Huruvida de endast beror på typen av sitt indata
  (detta är ett specialfall av rena funktioner)

(define (c? x)
  (if (a? x)
      #t
      (if (b? x)
          #t
          #f)))

(define (d? x)
  (= (+ 1 2 3) 6)
  (if (a? x)
      (b? x)
      #f))

(define (e? x)
  (not (a? x)))

c? = U(a?, b?)
d? = I(a?, b?)
e? = D(top, a?)

Is is always possible to reduce functions that only depend on the
types of their arguments to type expressions? If so, how? What exactly
are the benefits of doing that?


(macro
 (lambda (gensym form)
   (gensym)))

(define-macro (ten-times . x)
  (let ((loop-gs (gensym))
        (i-gs (gensym)))
    `(,let ,loop-gs ((,i-gs 10))
        ,@x
        (,if (,> ,i-gs 1)
             (,loop-gs (,- ,i-gs 1))))))

(define-macro (ten-times . x)
  (w/uniq (loop i)
    `(,let ,loop ((,i 10))
        ,@x
        (,if (,> ,i 1)
             (,loop (,- ,i 1))))))


Primitive operations in the language:
* lambda, procedure?
* call/cc, call/values
* make-macro, macro-procedure, macro?
* quote, symbol?
* cons, car, cdr, pair?, null?
* if, eq?

* make-type, make-colon, colon?, colon-car, colon-cdr
* var


Types:
* Is a tuple of
  * type|pure|unsure
  * The type
* Can be (?)
  * top
  * bottom
  * function types, parameter types -> return types
  * union types, U(A, B)
  * intersection types I(A, B)
  * difference types D(A, B)
  * primitive types.
* Operators
  * Equality, =
    * A = B iff B = A
    * A = (B = C) iff (A = B) = C
    * A = B and B = C => A = C
    * Two primitive types are = iff they are the same type predicate
    * Two function types are = iff the parameter types and return types are =
    * U(A, B) = U(B, A)
    * U(A, U(B, C)) = U(U(A, B), C)
    * I(A, B) = I(B, A)
    * I(A, I(B, C)) = I(I(A, B), C)
    * I(A, B) = A  iff  A <= B
    * U(A, B) = B  iff  A <= B
    * D(A, B) = bottom  iff  A <= B
    * A <= top
    * bottom <= A
  


Compile-time macro expansion, type inference and purity analysis step:
* Takes as input the lexical environment, which maps from names to
  * { (lazy) pure or impure } type
* Gives as output a partially macro-expanded and type and purity annotated s-expr.
* Iterates over the sexp. When it finds an invocation to
  * a pure macro, it macro-expands it and continues iterating
  * an impure macro, it gives up that subexpression
  * a non-macro, it continues iterating
  * undecided, it gives up that subexpression.
    Or should it optimistically attempt the analysis?
  * lambda, it ...

Attempt to define the algorithm on a pure language with currying and
see how it goes?

Do macro expansion by undoing the CPS transformation? *Or by saving
the direct style s-expr in the CPS code?*




(let ((a 3)) (+ a a))

AST: sexpr (yet to be processed), lambda, call, ref or val object
