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
* [Go Concurrency Patterns](http://www.youtube.com/watch?v=f6kdp27TYZs) (again,
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
3. You have the Go compiler, and toolchain, installed.

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

The `NewWorker` function does nothing more than create a new `Worker` object,
and return it. The lone argument to `NewWorker` is the buffered channel of
un-buffered, `WorkRequest` channels. In the creation of the `Worker` object, we
also giving the worker an un-buffered channel for it to receive the work
requests on. Truthfully, we really don't *need* this channel to be buffered.
The reason for this is that the worker can really only do one thing at a time,
and what we want here, is for the *dispatcher* to only give out work requests
to workers that are idle, and waiting for work.

The `QuitChan` is also un-buffered, only because it doesn't *need* to be. We
avoid the blocking nature of un-buffered channels in the `Worker.Stop` function
by wrapping the send in an anonymous goroutine; we will not block the call to
`Worker.Stop()`, but the worker will stop when we want it to.

### Step 4: The dispatcher

It is now time to implement our *dispatcher*! 

<script src="https://gist.github.com/nesv/9233300.js"></script>

Deceivingly simple, isn't it? At the top of the file, we have declared, and
initialized, our `WorkerQueue` which is the buffered channel that holds the
work channels from each worker.

> Remember, the worker is responsible for adding itself into the workers
> queue.

Within the `StartDispatcher` function (to which we provide the number of
workers we would like to start), we initialize the `WorkerQueue` channel with
a buffer size the same as the number of workers we are going to start.

Then, we create and start the workers. In the `NewWorker` function, the first
argument is an integer, which we use as a numeric ID for the workers, so that
we can see which worker is doing the work.

The final block of code, the anonymous goroutine, is what actually dispatches
the queued work requests. We pull a work request off of the `WorkQueue` channel
(which we declared and initialized in `collector.go`), then we then fire off
another, anonymous goroutine to  send the received `WorkRequest` object to the
*worker*.

The reason we send the work request to the worker in another goroutine, is
so that we make sure the work queue never fills up. Goroutines are wonderfully
inexpensive things; they are not threads, we can start as many as we want, and
the scheduler in Go's runtime will perform its namesake task. With this
approach, we can pull a work request off of the work queue, then send the work
request to a worker, but the `worker := <-WorkerQueue` will block.

It may seem silly, but because goroutines are cheap, we can care more about 
making sure the work queue never fills up, and that we give work to the workers
as soon as possible. We would rather block on receiving a work request, than
sending a work request to a worker.

## Step 5: Putting it all together

At this point, we have:

* A *collector* that receives, checks, and queues work requests
* A *worker* that does the work
* And a *dispatcher* that pulls work off of the work queue, and gives it to
  a worker
  
The last thing we need to do is tie it all together!

<script src="https://gist.github.com/nesv/9233955.js"></script>

The `main` function is the entry-point for Go, when you write an application.
We are also allowing the user to specify how many workers they would like to
run, and what address the HTTP server should listen on. Both of the
command-line flags are optional, and we provide sane defaults.

Since we took care to make things as simple as possible in the other files of
our application, all we need to do is start the *dispatcher*, register the
*collector* to the `/work` endpoint (remember, our collector was just an HTTP
handler function), and then start the HTTP server.

## Building the application

Each code snippet provided in this post can be put into its own file, and by
Go's coding standards, they should be in their own files, as each snippet
provides an functional piece of our application.

So, assuming you put each of these code snippets into their own file, you should
have:

* `work.go` which holds our `WorkRequest` struct
* `collector.go` which has our HTTP handler function, and also declares and
  initializes our work request queue `WorkQueue`
* `worker.go` which implements our `Worker` struct, its `Start` and `Stop`
  methods, and our convenience function `NewWorker`
* `dispatcher.go` which is where we implement our dispatcher, as an anonymous
  goroutine in the `StartDispatcher` function
* `main.go`, where we call the `StartDispatcher` function, register the
  `Collector` HTTP handler function, and start the HTTP server
  
To build the application (let's call it `queued`), run the following command:

	$ go build -o queued *.go
	
Now, to run our application, how about we start it with 2048 workers, just for
kicks?

	$ ./queued -n 2048
	...
	Starting worker 2047
	Starting worker 2048
	Registering the collector
	HTTP server listening on 127.0.0.1:8000
	
Sweet! Now, in another terminal window, let's write a little Bash one-liner, to
flood our collector with requests:

	$ for i in {1..4096}; do curl localhost:8000/work -d name=$USER -d delay=$(expr $i % 11)s; done
	
This one-liner, will loop for 2x the number of workers we defined, and upon
each iteration, it will call `curl(1)` to make an HTTP POST request to 
`localhost:8000/work` passing along our username, and a delay, which is
calculated as the modulo of our current loop iteration and the number `11`.

I chose the number `11` here, because it will allow us to send work requests to
the collector with delays from 0..10 seconds, which means we are also testing
whether or not our sanity checking works.

In the terminal window where we had `queued` running, you should see a flurry of
messages scrolling past. You may also notice, that the workers are running out
of order! This is because we employed a "first-come/first-serve" type of queue.

## In conclusion

Hopefully this article made sense, if it didn't (or if there is a typo, or error
somewhere), please feel free to hit me up on Twitter, or Google+. Heck, you 
could even create a new 
[GitHub issue](https://github.com/nesv/nesv.github.io/issues/new) if you'd like!
