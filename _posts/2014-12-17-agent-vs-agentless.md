---
layout: post
title:  "Agent vs. Agentless"
date:   2014-12-17 13:19:00
categories: devops configuration-management
---

There has been something kind of burning on my mind, lately, and it's mostly
[Ansible](http://ansible.com)'s fault.

> Ansible is an amazing tool, this post isn't harping on the quality of the
> project, in the slightest.

My biggest beef with Ansible (aside from the shaky promotion of variables), is
is how they market the project as being "agent-less". That's kind of a
crappy ploy. Have you ever tried to use Ansible on a system that does not
have Python installed (or has an ancient version of Python installed)? The
only sentiment I can offer is: good luck.

## Agent-less?

So, what does "agent-less" mean? Heck, what does it mean to have an "agent"?

An agent may make more sense if you prepend the word "execution": "execution
agent". With Puppet, you have the `puppet agent` command; Salt has minions,
and Chef has something similar. In these other tools, the agent handles
communication with the "master", for the purposes of getting contextual
settings, so that these settings can be applied to the system the agent is
running on.

I get the impression, from Ansible's marketing, that they merely celebrate 
the fact that you don't have to install something extra onto the remote 
hosts you want to manage. Again, this isn't entirely true. You basically need
an SSH daemon running (which, fine, almost every system ever has this
pre-installed), and an up-to-date Python interpreter. There are some cases,
where a Python interpreter just isn't installed on the to-be-managed node,
by default (e.g. Arch Linux, OpenBSD). The argument is that you could use
Ansible's `raw` module to install Python, prior to running any other steps,
but then you lose the niceness that comes from using the package-specific
modules, like `apt`, `yum`, `pacman`, etc.

## So, agents then?

If you'd like. Essentially, this all boils down to personal choice. Again,
I would like to stress that I am not saying Ansible is a sub-standard product,
in any way, shape, or form. For configuration management, it is definitely
my go-to tool.

One of the big headaches with agent-oriented tools is setting up that
master-minion relationship. You can set up the master, but then you have to,
somehow, set up the minions so that they know about the master. In situations
like these, I think a laughably-accurate solution would be to use Ansible to
install Salt, or Puppet.

## What would make you happy, Saika?

Here is my wishlist:

*	No more Python, or Ruby. Managing the dependencies for these is
	ghastly. How about no interpreted languages, at all?
*	Gimme a statically-compiled binary. In the worst-case scenario, all
	I would have to do is use `scp(1)` to copy, say a `.tar.gz`, to the
	to-be-managed node. This would also mitigate needing to have an
	up-to-date interpreter installed.

That's it, really; two items. I don't give a hoot if it the tool has a
master-minion architecture with agents, or it is is agent-less.

I know it really cuts down on the development time to write these tools in an
interpreted language, and you can do lots of nifty things like dynamically
import other packages/modules/libraries, but you still run into the
pain-in-the-ass situation of having to manage dependencies. For example, let's
say you want to use tool *X*. It has a dependency on json-module-1.2.3, but
your operating system-du-jour comes with json-module-0.2.4. You may try to
build your own package of json-module-1.2.3, but most often, you will find
that a crap-ton of other system packages **rely** on json-module-0.2.4, for
stability's sake. You pretty much have to resort to doing something creaky,
or downright hackish just to get things working.

Ultimately, let's write something in Go, C, C++, or in any other language that
can compile down, into a static binary, where the only run-time dependency, is
that the system be powered on.

## In closing

I have been stewing on this for a while, and started a GitHub repo: 
[govern](https://github.com/nesv/govern). I honestly haven't puch much effort,
or time, into the project, and the only explanation is that I have other things
to tend to (work, family, friends, etc.).

Should you, dear reader, be interested in working on *govern*, I would
whole-heartedly welcome it.

> **NOTE**
>
> Everything that is currently in the *govern* repository can be trashed. It
> has really only served as a bin for my thoughts, as I postulate on my ideal
> configuration management system.

End of transmission.
