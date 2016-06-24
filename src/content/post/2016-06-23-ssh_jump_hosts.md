---
date: 2016-06-23
title: SSH jump hosts
categories:
    - today-i-learned
tags:
    - SSH
    - networking
---

Recently I learned about SSH jump hosts. Jump hosts make it easier to
connect to an SSH server via an intermediate hop—i.e., connecting to an
SSH server and, from there, connecting to yet another SSH server.

For years I did this “jumping” manually, by entering and running two
distinct SSH commands. E.g.,

```sh
craig@myclient$ ssh proxy_host

Welcome to proxy_host!

craig@proxy_host$ ssh target_host

Welcome to target_host!

craig@target_host$ echo "Now I can do some work."
```

This is cumbersome, even when using a private–public key pair to
eliminate passwords. This is where jump hosts help.

A jump host allows me to run one `ssh` command with the same effect __as
if__ I had run the two `ssh` commands above:

```
craig@myclient$ ssh target_host

Welcome to target_host!

craig@target_host$ echo "Now I can do some work."
```

SSH sets up the proxy connection behind the scenes. But you need to
configure the jump host in your client's `.ssh/config` file. Here's
mine:

```
Host target_host
ProxyCommand ssh -q proxy_host nc %h %p
```

When I run the command ``ssh target_host``, what SSH does is to connect
to ``proxy_host`` and then to use `nc` to forward the connection to
``target_host``. SSH expands the `%h` and `%p` fields to be the target
host and port—in this case, `target_host` and `22`. This expansion
allows you to configure multiple targets in your `.ssh/config` via
wildcards.

```
Host *.example.com
ProxyCommand ssh -q proxy_host nc %h %p
```

This stanza will work for using ``proxy_host`` as a jump host for
connecting to, say, `foo.example.com` and `bar.example.com`. Wildcards
are helpful if you're jumping into a large private network with a lot of
boxes all within one domain.
