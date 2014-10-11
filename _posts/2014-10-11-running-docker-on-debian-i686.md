---
layout: post
title: Running Docker on Debian i686
date: 2014-10-11 13:04:22.000000000 -08:00
status: published
type: post
---
*Update 2014-10-11*: I've updated the post for new versions of Docker
packaged with Debian, and I've included steps to get it running on
32-bit hosts as well.

I want to get Docker running on my netbook. From the research I've done,
it appears that running Docker on 32-bit systems is officially unsupported.

Who doesn't draw outside the lines once and awhile?

First up, get the Debian "Testing" repository (codename: "Jessie") configured
and update your package list. You'll also need to pin the repository so that
there aren't any accidental upgrades of unrelated packages.

{% highlight text %}
$ cat > /etc/apt/sources.list.d/debian-testing
deb http://ftp.debian.org/debian testing main
deb-src http://ftp.debian.org/debian testing main
^D
$ cat > /etc/apt/preferences.d/debian-testing
Package: *
Pin: release a=testing
Pin-Priority: -100
^D
{% endhighlight %}

Next, install the `docker.io` package from the new repository. If there
are any broken or missing dependencies (ie: `libdevmapper1.02.1`) then prepend
them to the package list as well:

{% highlight text %}
$ sudo apt-get install libdevmapper1.02.1/testing docker.io/testing
{% endhighlight %}

To be able to operate Docker without superuser privileges, I needed to be a 
member of the `docker` group. You'll need to log back in for the changes to
take effect.

{% highlight text %}
$ sudo gpasswd -a administrator docker
Adding user administrator to group docker.
{% endhighlight %}

I use LVM on my netbook; I've allocated roughly 90% of the available disk
space to `/home`. However, Docker will need significant amounts of disk
space to store images and it stores them to the root partition where there
isn't much disk space available. To solve this problem, I set up a bind mount
from `/home/docker` to `/var/lib/docker` and added it to `/etc/fstab` so it
will be mounted on boot.

{% highlight text %}
$ sudo service docker stop
$ sudo mkdir /home/docker
$ sudo rm -rf /var/lib/docker/*
$ echo "/home/docker /var/lib/docker none bind 0 0" | sudo tee -a /etc/fstab
$ sudo mount /home/docker
{% endhighlight %}

Docker's official build disables the AUFS storage driver but it still appears
to be the default with Debian. I'd like to use the "device-mapper" storage
driver instead; I've added `-s devicemapper` to the `DOCKER_OPTS` variable
in `/etc/default/docker`.

Now we can start Docker:

{% highlight text %}
$ service docker start
{% endhighlight %}

However, there's one problem remaining: Docker doesn't support 32-bit hosts.

When you try to pull an image from the Docker registry and run it, you'll 
encounter a cryptic error message: `exec format error`. You'll see this error
message when the operating system doesn't understand the kind of executable
you're asking it to run.

This occurs because the images published to the registry are for 64-bit hosts
only, and because we're running a 32-bit host, the operating system doesn't
know how to run the 64-bit executable.

Thankfully, we can solve this problem by building our own base image instead:

{% highlight text %}
$ sudo debootstrap wheezy /tmp/wheezy_base32_rootfs/
$ cat > /tmp/wheezy_base32_rootfs/etc/apt/sources.list
deb http://ftp.us.debian.org/debian stable main
deb http://security.debian.org/ stable/updates main
deb http://ftp.us.debian.org/debian/ stable-updates main
^D
$ sudo tar -czf /tmp/wheezy_base32_rootfs.tgz -C /tmp/wheezy_base32_rootfs/ .
$ cat /tmp/wheezy_base32_rootfs.tgz | docker import - base32
{% endhighlight %}

Now we have our own 32-bit base image and Docker can use it:

{% highlight text %}
$ docker images
REPOSITORY      TAG     IMAGE ID        CREATED         VIRTUAL SIZE
base32          latest  65b5e2e55c7f    31 seconds ago  223.5 MB
$ docker run -i -t base32 /bin/bash
root@460accd6fab3:/#
{% endhighlight %}

We're done.
