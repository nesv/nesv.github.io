---
layout: post
title:  "Two-legged OAuth 1.0 calls, in Go"
date:   2013-09-17 21:41:00
categories: oauth golang
---

Half-way through last week, I stumbled across [context.io](http://context.io)
and thought it would be pretty decent to write up a nice, small API client
for [Go](http://golang.org)! I am going to skip over what exactly it is that
context.io does; I provided a link above so you can check it out for yourself.

After signing up, I started to read through their documentation and discovered
that they use OAuth to authorize any API requests. Cool beans! Yeah, well, first
of all, it's OAuth 1.0. Secondly, it's *two-legged OAuth 1.0*.

### A brief explanation of OAuth 1.0

What is "two-legged OAuth" you ask? Well, to *really* simplify it, OAuth is
typically a "three-legged" authentication, and authorization procedure. When
you go to enter into a conversation with an OAuth "provider" your application
must have two things in its possession: a consumer *token* and a consumer
*secret*. In the realm of shared-key encryption schemes, you pass around your
token to vouch that a message came from you, but you keep the secret to
yourself, as it is used in encrypting your communications, as well as allowing
you to definitively say "yes, this message is from me".

> In the OAuth scheme of things, your application is what is referred to 
> as the "consumer".

The first leg of the OAuth conversation is when you approach the
*service provider* (in this case, context.io) with your consumer token and 
secret, you are effectively asking to be authenticated; you are proving to the
service provider that you are who you say you are.

The second leg of the conversation - assuming your initial authentication
request succeeded - is the service provider responding with your *access token*
and *access secret*. These are similar to your consumer token and secret,
however, they are temporary and will expire after a certain amount of time.
The access token and secret are used to *sign* and encrypt your subsequent
requests to the service provider.

The third, and final, leg of the OAuth conversation is any request you make
to the service provider, with your access token and access secret to
authorize your request, before the access token and secret expire.

> Expiry times on access tokens vary by service provider.

### So, what is two-legged OAuth 1.0 then?

Armed with that knowledge, you are probably asking, "well, which leg is *not*
used in two-legged OAuth?" The answer to that, is "the first one". The initial
authentication request is "skipped" because you are assumed to already have
your access key/token and secret.

I feel it necessary to inform you, dearest reader, that this wasn't an "out of
the box" process. I cannot remember how I started trying to figure this out,
but I think it involved looking at how the 
[rauth](http://rauth.readthedocs.org/en/latest/) Python library handled such
things, mixed with various snippets of things I had come across online. The
final solution to this issue could not have been reached without a teeny-tiny
patch to the OAuth 1.0 package I used:
[github.com/mrjones/oauth](https://github.com/mrjones/oauth).

> I just want to gush for a moment: I love the fact that I was able to make
> the small change necessary to make this work, and that the
> [maintainer](https://github.com/mrjones) of the library merged in my
> pull request so shortly after I made it. Open development is wonderful.

Here is a working example of making a two-legged OAuth 1.0 call:
{% highlight go %}
package main

import (
    "fmt"
    "github.com/mrjones/oauth"
)

const (
    Key    = "4AxhC3QDTFPJXSUE"
    Secret = "v6U7Dvz2T5jtKzdmUEZPDWrtAjA5MGNR"
)

func main() {
    consumer := oauth.NewConsumer(Key, Secret, oauth.ServiceProvider{})
    accessToken := &oauth.AccessToken{}
    response, err := consumer.Get("http://some/remote/endpoint", nil, accessToken)
    fmt.Println("Response:", response.StatusCode, response.Status)
}
{% endhighlight %}

I am not going to explain most of this code, because I am assuming that you
already know your way around the Go programming language, and that you are
here to learn about making two-legged OAuth calls, in Go. It is rather
*apropos*, but if the above code sample is leaving you baffled, you should
*really* work your way through the [Go tour](http://tour.golang.org), and
then read [Effective Go](http://golang.org/doc/effective_go.html).

That aside, here are the strange things about the code above:

1. We already know our access token (also referred to as a "key" in some
   cases), and access secret, however we are providing it to the
   ``oauth.NewConsumer()`` function, which we would typically feed our
   *consumer* token and secret into
2. We are not specifying the service provider's URLs that would typically
   tell the library where to fetch our request, authorization, or access 
   tokens (we are leaving them all blank)
3. We are leaving our request's access token and secret blank

In conclusion, this works, and I am not entirely sure how. Additionally, I
would like to thank [Matt Jones](https://github.com/mrjones) for having done
most of the hard work by writing an OAuth library for Go, and also for 
responding so quickly to my pull request.
