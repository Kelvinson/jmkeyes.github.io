---
layout: post
title: Running Docker on Debian
date: 2013-11-28 10:39:22.000000000 -08:00
status: published
type: post
---
I've been wanting to get Docker running for some time. The only
remaining hurdle was the age of the Debian's stock kernel, which
was too old to support the Linux container infrastructure. Now
that the kernel has been updated, installing docker is simple.

Add the Debian "testing" repository to your APT `sources.list`:

{% highlight bash %}
$ cat > /etc/apt/sources.list.d/debian-testing
deb http://ftp.debian.org/debian testing main
deb-src http://ftp.debian.org/debian testing main
^D
{% endhighlight %}

I recommend using pinning so that all testing packages from the
testing repositories will only be installed when requested:

{% highlight bash %}
$ cat > /etc/apt/preferences.d/debian-testing
Package: *
Pin: release a=testing
Pin-Priority: -100
^D
{% endhighlight %}

Now update the list of packages and install Docker:

{% highlight bash %}
$ apt-get update
$ apt-get install docker.io/testing
{% endhighlight %}

Done.
