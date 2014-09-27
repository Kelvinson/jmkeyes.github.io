---
layout: post
title: Automatically Rewrite Git Push/Pull URLs
date: 2013-04-06 17:08:59.000000000 -07:00
status: published
type: post
---
Git can automatically rewrite a repository URL if it matches any filter
specified in it's configuration file. These filters operate during both
sending and receiving.

The following addition to your global ".gitconfig" file will convert all
HTTP(s) repository URLs, as well as those prefixed with "github:" to use
the Git URL scheme during push and pull operations:

{% highlight ini %}
# ~/.gitconfig
[url "git://github.com/"]
    insteadOf = https://github.com/
    insteadOf = http://github.com/
    insteadOf = github:

[url "git+ssh://github.com/"]
    pushInsteadOf = https://github.com/
    pushInsteadOf = http://github.com/
    pushInsteadOf = git://github.com/
{% endhighlight %}

Now you can clone GitHub repositories using the "github" prefix:

{% highlight bash %}
$ git clone github:username/repository
Cloning into directory ...
Receiving objects: 100% (1/1), done.
$
{% endhighlight %}

Git will now rewrite any pull to Github to use the bare Git protocol
and push to Github to the SSH-secured repository address.
