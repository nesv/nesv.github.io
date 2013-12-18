---
layout: post
title:  "REST API clients are too complicated"
date:   2013-11-05 18:00:00
categories: programming
tags: rest python golang
---

I have a serious bone to pick with most REST API clients out there. Not that they are of poor quality, or anything, but rather their execution is what bothers me.

In any given situation where I would like to consume a web service, I need to not only become familiar with the service itself, but then hope that there is a *decent* client/driver in the programming language I am working in at that moment. In some cases, you may have multiple available clients, but then it comes down to recommendations, and failing those, general, first-hand testing to see how that client fits into your existing code, or whether the client works (if it even works) or &mdash; the worst situation &mdash; just how much of it works.

Furthermore, how long is this interim driver going to last? Is it going to fall apart when the upstream service provider decides that they have to make a hard cut-over to the next version of their API (no deprecations)?

### It's in the take-off

First things first, let's take an API: [ContextIO](http://context.io). It is an awesome service that converts all of your email into JSON structures, for easy, programmatic use.

The API is clean-cut, makes sense, and is not overbearingly large. Any developer who wants to learn how to interact with it, is going to read their REST API documentation first, in the hopes to learn the semantics the API, as most often, the clients for that API service carry on, and expose those same semantics, and structure of data.

More often than not, though, the logical progression of data in the service provider's API does not match up with that of the client's; the factors leading to this are usually due to the design of the programming language the client is for. JSON maps ever-so-nicely to Ruby hashes, and Python dictionaries, because both of those languages are dynamically-typed; more often than not, however, people try to unmarshal the received data into classes. In the event of strongly, or statically-typed languages (like Go, for example) you end up losing a lot of execution speed because you start delving into reflection facilities to guess at what type the data in the value field portion of JSON key-value pair is.

> In Go, you have the `interface{}` type, which allows you to hold data of an unknown type, but trying to coerce a value of out an `interface{}` is slow.



### old

A few months ago, I started working on [gothub](https://github.com/nesv/gothub),a GitHub client for the Go programming language. It started turning into a mental exercise of charting out which methods get pinned to which structs, in the "most-sensible" way. Life, and other such things got in the way, and it is much to my chagrin that the project (along with several others) have fallen by the wayside.

More recently, I found out about [ContextIO](http://context.io) &mdash; they are an absolutely wonderful service that converts your email into JSON objects, and makes it worlds easier to work with, in a programmatic way. (If you have done *any* sort of work involving IMAP interactions and email message parsing, you would definitely appreciate how much nicer it is to use the service they provide, than to try and write another message parser/IMAP library.

Since I have been really digging [Go](http://golang.org) for the past, few years, I noticed that ContextIO did not provide a Go client for their API. I was not really expecting them to be "trendy" enough to provide a Go client; in fact, their [list of official clients](http://context.io/docs/2.0/libraries) is fairly small, but are for the more-popular languages (like Python, Ruby, PHP,
etc.). Thus, I set out to write a ContextIO client for Go.

> I know, I know. Shame on me for starting another project while I have a few sitting in GitHub repos, unfinished.

## L'&eacute;piphanie

Somewhere in this venture, I had an epiphany. Perhaps, it was a culmination of seeing how many other Go developers write their idiomatic client libraries for interacting with other services; the Go community &mdash; for the most part &mdash; likes to keep the presentation of a package similar to other packages that perform similar functions. For example, there are many packages that
are described as having "print-like" call signatures, for the functions provided by that package. If you look at the [`fmt`](http://golang.net/pkg/fmt) package, you will notice that most of the `*print*` functions take a variable-length list of arguments (these functions are said to be *variadic*).

Now, as far as in-memory, key-value stores go, [Redis](http://redis.io) is my absolute favourite; I would use it before I use memcached, or Riak, any day of the week. If you care to look at the [redigo](https://github.com/garyburd/redigo) package by Gary Burd (a Redis client for Go), you may notice that there is really only one function that you ever need to call to issue a command to the Redis server: `*Client.Do()`. You may then also notice, how it provides you with a print-like interface.

So, between the `fmt` package in the Go standard library, and the `garyburd/redigo` package, we are seeing two packages that you interact with in a similar fashion, yet they server different purposes at first glance. If you delve into the approach a little further, you will come to realize you are just sending a string out, over an I/O stream, in both cases.

The epiphany I had was most likely some strange combination of the mental stress of trying to figure out how to eloquently associate methods to structs to create an object hierarchy, and the notion of using print-like, variadic functions.

> What I have noticed as being the "rule of thumb" for "when to use variadic, print-like functions" is whenever you are constantly having to join strings together, with a common seperator.

## How this applies to REST API clients

Most REST API clients I come across make too much of an attempt to try and map the REST API to the object-oriented facilities of the language the driver is being written in. This may be something to blame Java for, but that is a topic for a seperate discussion.

To use the GitHub API as an example, if you want to see the branches in a repository, you would have to issue the following HTTP request (assuming you already have an auth token): `GET https://api/github.com/repos/:owner/:repo/branches`.

Trying to map this out &mdash; from just this little snippet alone &mdash; you would probably want to organize your data thusly:

1. Owner
2. Repository
3. Branch

Yes, this makes sense, and is not a terribly complicated situation. Where it starts to get complicated, however, is when you start trying to associate methods to delve deeper into the data tree. There are many situations (not just in the GitHub API) where the data types you want to access are reachable from more than one path. In those cases, you end up having to try and juggle the permutations of data-descension paths.

So, how can you avoid this nightmare? The approach will slightly differ, depending on the typing strictness of the language you are using.

### Strongly-typed languages

Using Go as my poster child for a strongly-typed language (and for the sake of this example), the first thing I did was create all of the structs that I would be unmarshaling the response data to. In both cases &mdash; the GitHub, and ContextIO APIs &mdash; we are receiving JSON data. This *greatly* simplified everything by completely eliminating the need to trying to figure out how to elegantly pin functions to structs, to delve down into the next level of the API ("data-descension").

All the client does, really, at this point is worry about establishing the connection to the service provider, parsing out the desired data from the responses, and mapping this data to the structs we have defined.

A typical (overly-simplified) call may look like:

	var owner User
	...
	var repo Repository
	...
	var branches []Branch
    err := client.Do(&branches, "GET", "repos", user.Username, repo.Name, "branches")
    
> I am fully-aware that this example does not allow for issuing POST, PUT, or PATCH requests that would take a request body, URL-encoded parameters, or even custom headers. (read: "overly-simplified")

So, in this little snippet, we have initialized a `User` object, and a `Repository` object. Then, we gather an array of `Branch` objects. Without getting into the nuances of Go's syntax, the `client.Do()` call is letting us specify the HTTP method of the call ("GET"), the URI that will provide us our list of branches (broken up into the parts that are seperated by slashes), and then a pointer to the initialized object that will hold the received data (`&branch`).

In this case, the `client.Do()` function is responsible for:

- building the URL from the specified URI parts
- issuing the HTTP call (using the specified HTTP method)
- unmarshaling the received data into the specified struct so you can easily consume it

### Dynamically-typed languages

The big difference is most dynamically-typed languages (like Perl, Python, Ruby, and PHP) is you can leave out the part of defining your custom data types ("structs"). In these cases, I highly recommend we as developers strive to do as little as possible, and just rely on some of the container/collection types provided to us.

In Python, Ruby, Perl, and PHP, they all have concepts of a map type: in Perl and Ruby it is called a "hash", in Python a "dictionary". I will stipulate why I suggest using these types in the next section.

Without having to worry about writing a slew of custom data types now, we can just focus on the part of the client that establishes a connection to the service provider, and makes requests. Here is a Pythonic analog to the Go snippet, above:

	user = client.do(...)
	...
	repo = client.do(...)
	...
	branches = client.do("GET", "repos", user.username, repo.name, "branches")
	
In this example, the `client.do()` method is doing the exact same thing as it would in the Go example, the only differences in the examples are due to the idioms in each language. With this Python example, I assume this sample block would be wrapped in a `try/except` block, and the `client.do()` method would be responsible for raising any exceptions.

## Why this approach?

### Implementation

For starters, it is *much* simpler to implement (more so, for the dynamically-typed languages). All you really have to worry about is establishing a connection to the service provider, making the requests, and ensuring that you can map the service provider's data to the native data types provided by your language-du-jour.

### Maintenance

Now, in the *Dynamically-typed languages* subsection, I said I would explain why I favour map types (over custom classes, for example): **maintenance**. If the service provider starts to change the structure of the data they are sending you, yes it may require you to slightly modify your application to work correctly, but the part of you that wrote this library won't have to change a thing! The library, at this point, has turned into nothing more than some boilerplate code to connect to a specific service. The strongly-typed languages may have to make some modifications though. The overall motto here is to get the provider's data into your language's native types as quickly as possible, so that it is super-convenient for you (as the developer of the library) and whomever your library is intended for (maybe you, as well).

### Less abstraction

Now, my favourite reason for this approach, is that it removes the level of seperation/abstraction between the person using the library, and the service provider. More often that not, if you look at a REST API library, you will notice that the library is often out of date for (hopefully) a short while, after the service provider updates their API. In those cases, people are harping on you (or sending you pull/merge requests) to get this up and running again, but now **two** APIs have changed: the service provider's, and yours.

I can appreciate the notion of wanting to make someone else's life easier through use of your client, but I also don't want to be (or for you to be) an interim point of failure. Why not simply remove yourself from the data-interpretation-mix? You are creating an API for the API &mdash; stop it. Instead, provide the convenience through writing that little bit of boilerplate that allows people to connect to a service from their programming language of choice. Connection schemes rarely change with APIs.

This also forces the service providers to keep their documentation up to snuff. Do not look at this as shifting documentation responsibility onto the service provider and away from you (in the currently-common approach of API clients). You could not write your API client if the service provider hadn't written their documentation, so just distribute the boilerplate code, and refer your users to the service provider's documentation; "teach them to fish".