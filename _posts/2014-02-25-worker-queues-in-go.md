---
layout: post
title: "Writing worker queues, in Go"
date: 2014-02-25
categories: golang
---

Have you ever wanted to write something that is highly concurrent, and performs
as many tasks as you will let it, in parallel? Well, look no further, here is a
guide on how to do just that, in [Go](http://golang.org)!

### This isn't new

For an absolutely riveting (to me) talk on concurrency patterns, I highly
recommend watching the following videos:

* [Concurrency is not Parallelism](http://vimeo.com/49718712) by Rob Pike is a
  good video to start with, as it imparts the theory behind using concurrency
  to write applications that execute tasks in parallel. This is definitely
  my favourite video in the bunch.
* [Go Concurrency Patters](http://www.youtube.com/watch?v=f6kdp27TYZs) (again,
  by Rob Pike) gives more concrete examples of how to employ various
  concurrency...erm...patterns, in Go. It also provides a comparison on how
  Go's concurrency model differs from those in other languages, like Erlang.
* [Advanced Go Concurrency Patterns](http://youtu.be/QDDwwePbDtw) by Sameer
  Ajmani takes *Go Concurrency Patterns* and extends on it, by showing you
  even more patterns you can employ in your code.
  
If all you want is a basic understanding of concurrency, then you could get
away with only watching the first video.

### Some assumptions

While this will not be covered in the code examples, we are going to make
several assumptions about our system:

1. Under no circumstances will the clients issuing work requests wait for the
   work request to finish. This is simply to avoid over-complicating our
   examples.
2. A work request will take a person's name, and a length of time by which to
   delay the printing of that person's name. The length of time must be
   parseable by `time.ParseDuration()`, and must be between 1 (one) and 10
   (ten) seconds, inclusively.

Hopefully, by the end of this article, you will be able to figure out how we
can add in support for clients to wait for a work reqeust to finish. I also
hope to leave you with an example that is easy to extend beyond your wildest
imaginations. :smileyface:

### Some terminology

There are a few terms we are going to be using, in this article, to describe
the various parts of our queueing system.

A **collector** is going to be responsible for receiving work requests, and
adding them to the *work queue*.

The **work queue** is, in Go terms, a buffered channel, which just lets work
requests collect. The *work queue* is buffered, so that we do not block the
collector.

Our **dispatcher** are responsible for pulling work requests off of the queue,
and distributing them to the next available *worker*. To keep things clear,
and voodoo-free, we are going to have our *dispatcher* maintain several queues,
the first of which, being the *worker queue*.

The **worker queue** is the weirdest part about all of this (if you are not
familiar with Go, and maybe, even if you are). It is a buffered channel of
channels. The channels that go into this channel, are what the *workers* use to
receive the work request. If this does not make sense now, it probably will as
we implement the system.

Lastly, **worker**s are responsible for performing a unit of work. In our
examples, we are going to make workers responsible for letting the *dispatcher*
know when they are ready to accept more work.

### Step 1: Defining our work request structure

We need to be able to send our work request to the workers, via the
*dispatcher*. Now, due to Go's strict type system, channels must be typed. Yes,
channels are a type, but they are a type that is used to send values of *other*
types around. To satisfy this behaviour, we are going to create a `struct` that
holds our work request.

<script src="https://gist.github.com/nesv/9219467.js"></script>

### Step 2: The collector

The collector is nothing special; it receives client requests for work, builds
a work request that the workers can understand, and pushes the work onto the
end of the *work queue*.

Ideally, the collector should be responsible for running sanity checks on the
incoming work requests, and alert the client if their work request does not fit
within whatever acceptable boundaries you define. We also do not want our
collector to hold an open network connection for any longer than it has to.
Again, this imposition is in the name of keeping things simple.

Our collector is going to be a simple, HTTP handler function that we can 
register with Go's default HTTP server.

<script src="https://gist.github.com/nesv/9219811.js"></script>

Now, in this snippet of code, we have `WorkQueue` defined as a channel that
can be used to send `WorkRequest` objects around on, and it has a buffer size of
`100` (one hundred).

> The buffer size of the channel is completely arbitrary, but you want to set
> it high enough so that sending work requests over it does not fill up, and
> block the send operation: `WorkQueue <- work`.

### Step 3: The worker

Now, we need to implement a worker. What the worker needs to have, is a channel
of its own that the *dispatcher* (which we will implement next) can use to give
the worker a `WorkRequest`. We are also going to give our workers a numeric ID,
so that we can see which worker is performing the work.

<script src="https://gist.github.com/nesv/9220339.js"></script>
