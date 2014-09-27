---
layout: post
title: Setting Up Golang On Debian
date: 2014-10-10 20:48:22.000000000 -08:00
status: published
type: post
---
I recently started trying out [Packer][packer-io] for creating
deployable VM templates. It's lean and very flexible. However,
I'd also like to create plugins for it, which means diving into
a language I've never touched before -- Golang.

After enabling the Debian "Testing" [repository][debian-testing],
installing it was fairly easy:

{% highlight bash %}
$ apt-get install golang/testing
{% endhighlight %}

In order to use the Golang runtime but still download packages to
my home directory, I needed to set a few environment variables in
my `.bashrc`:

{% highlight bash %}
# ~/.bashrc
# ...

# Set GOROOT to the Go libraries themselves.
export GOROOT="/usr/lib/go"
# Set GOPATH to use ~/.go, then $GOROOT.
export GOPATH="${HOME}/.go:${GOROOT}"

# Also, modify $PATH to get access to the Go binaries.
export PATH="${PATH}:${HOME}/.go/bin:${GOROOT}/bin"
{% endhighlight %}

I then created the `.go` folder in my home directory. Any projects
retrieved with `go get` command will download to the first available
folder in `$GOPATH`, and I still retain access to the distribution
runtime libraries as well.

{% highlight bash %}
$ go get github.com/vmware/govmomi/govc
$ govc version
govc version 0.0.1-dev
{% endhighlight %}

Now it's time to create a better `ovftool` post-processor.

  [packer-io]:      http://packer.io/
  [debian-testing]: {% post_url 2013-11-28-running-docker-on-debian %}

