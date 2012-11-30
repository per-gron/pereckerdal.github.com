---
layout: post
title: The awesomeness that is libdispatch
tags:
- concurrency
- libdispatch
- language
- design
---

One of the greatest revelations that I've had the in last year is that [libdispatch](http://libdispatch.macosforge.org) (aka [GCD](http://en.wikipedia.org/wiki/Grand_Central_Dispatch)) is *awesome*. Few people seem to realize it. In fact, I'm not even sure that the authors of libdispatch understand how awesome it is.

The purpose of this post is to explain to you, my wonderful audience, why this is the case.

Discussions about libdispatch and GCD tend to be centered around details, like how one codes a timer or I/O, or how Objective-C blocks are great (or awful, depending on who you ask). These are interesting topics to talk about, but in my opinion you miss the forest for the trees if you focus on them.

The thing about libdispatch that really excites me is *composability*. I will explain what I mean by this, but first I need to give you some background.


### Concurrency

Concurrency is a Really Hard Problem, and everyone knows that. There is a plethora of different programming models for dealing with concurrency and parallelism. Here are some of the most common:

* **No concurrency**: C without threads or doing fancy stuff with processes follows this model.
* **Traditional lock-based shared memory concurrency**: C with pthreads or Java are examples of this.
* **[The actor model](http://en.wikipedia.org/wiki/Actor_model)**: Erlang is the most prominent example, but there are others, for instance [Rust](http://www.rust-lang.org).
* **Single-threaded cooperative concurrency**: [Node.js](http://nodejs.org) exemplifies this model.
* **[OpenMP](http://en.wikipedia.org/wiki/OpenMP)**
* **[Software transactional memory](http://en.wikipedia.org/wiki/Software_transactional_memory)** aka <abbr title="Software Transactional Memory">STM</abbr>: Originally implemented in Haskell, but is now implemented in several languages, most notably [Clojure](http://clojure.org).

I believe libdispatch is in a category of its own. Most people seem to think of it as some kind of add-on to a traditional lock-based model. You can do that and write useful programs, but if you do, you're missing out.


### Composability

The creators of STM argue that one of its main selling points is that [it enables arbitrary atomic operations to be composed into larger atomic operations](http://en.wikipedia.org/wiki/Software_transactional_memory#Composable_operations), which is impossible with lock-based programming. To quote the authors:

<blockquote>
  Perhaps the most fundamental objection [...] is that <i>lock-based programs do not compose</i>: correct fragments may fail when combined. For example, consider a hash table with thread-safe insert and delete operations. Now suppose that we want to delete one item A from table t1, and insert it into table t2; but the intermediate state (in which neither table contains the item) must not be visible to other threads. Unless the implementor of the hash table anticipates this need, there is simply no way to satisfy this requirement. [...] In short, operations that are individually correct (insert, delete) cannot be composed into larger correct operations.
  <div style="text-style:italic">—Tim Harris et al., "Composable Memory Transactions", Section 2: Background, pg.2</div>
</blockquote>

Lack of composability is the core reason why concurrent programming with locks doesn't scale.


### libdispatch

libdispatch deals with *queues* and *tasks*. Tasks are pieces of code that are scheduled to run on a particular queue. Queues execute tasks sequentially in the order they were added.

In practise, this makes programming with libdispatch feel like lock-based programming turned inside-out: With lock-based programming, you are concerned with making sure that only one thread accesses a certain resource at a time, while the way to think in libdispatch is that you ensure that a certain resource is only ever accessed by one particular queue.

In my experience, Objective-C code that uses libdispatch usually ends up in such a way that most code is non-thread-safe, and a few key classes explicitly support concurrency. These classes have an *objectQueue* and a *delegateQueue* (if the object has a delegate). Every public method begins with checking if the current queue is the objectQueue. If that's not the case, the method schedules its own execution on the objectQueue. Also, delegate messages are sent on the delegateQueue.

CocoaAsyncSocket is a fantastic library that [beautifully implements this concept](https://github.com/robbiehanson/CocoaAsyncSocket/wiki/Intro_GCDAsyncSocket).

At first, I didn't think much of this, but then it dawned on me that with this programming model, it is

1. easy to exploit concurrency where you need it; just create a new objectQueue for things that you want to be executed concurrently,
2. easy to reason about; as long as every public method of the class ensures that logic is performed on the objectQueue, and that delegate messages are sent on the delegateQueue, the class is thread safe, simple as that,
3. and it is composable! If you have two objects with some invariant that needs to be maintained, just make sure they have the same objectQueue.

This, my apparently captivated audience, is why I feel that libdispatch is awesome. It seems like practical use of libdispatch is able to capture many of the performance benefits of a shared memory model while maintaining a programming model that actually scales.


### When is this useful?

Like any programming model, libdispatch is not perfect. It does not replace any of the existing ones, but I think it is an interesting and useful addition to the toolbox. If you need shared memory, but don't need to have complete control over everything, it might be your best option.


### A final side note

Azul systems have developed a garbage collector for the JVM called [C4](http://www.azulsystems.com/technology/c4-garbage-collector). Its specs are astounding: It is capable of collecting garbage at rates of several GB/s on heaps with several hundred gigabytes of live data, all with virtually no GC pauses. Its main drawback is that it makes all programs slower, by up to 30% if I remember correctly, but for many applications that is a small price to pay for massive parallelism and soft-realtime GC.

I have wondered why C4 hasn't gained wider adoption than it has. There could be several reasons for this, but there is at least one show stopper: The relevant available models for concurrent programming simply don't scale as well as C4: With models that don't share memory, C4 is unnecessary; lock-based programming doesn't compose, and I would expect current STM implementations to break down on C4's scale.

As I see it, libdispatch's composability with shared memory could be a programming model that works great with C4.
