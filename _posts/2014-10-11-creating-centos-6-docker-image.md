---
layout: post
title: Creating a CentOS 6 Docker Image
date: 2014-10-11 15:58:22.000000000 -08:00
status: published
type: post
---
In my [previous post][1] I created a 32-bit Debian Wheezy image for use in
Docker. As much as I like Debian, I also use CentOS and RedHat family
distributions as well.

Both Debian and Ubuntu make the creation of chroot environments extremely easy
to do: just `debootstrap` with your release codename and wait a bit.

The process for CentOS, however, isn't well documented. There are a few
utilities out there to generate a fresh installation but all of the utilities
I tried either didn't work at all or were simply too complicated to use.

How hard could it be to make a CentOS chroot from scratch? Let's find out.

On the host Debian machine, you'll need the `yum` package installed:

{% highlight text %}
$ sudo apt-get install yum
{% endhighlight %}

We'll need to create the chroot directory, bootstrap an empty RPM database
within the chroot environment, and then install the basic packages we need
for a CentOS box.

{% highlight text %}
$ export BASEURL="http://mirror.centos.org/centos-6/6/os/i386/Packages/"
$ sudo mkdir /tmp/sysroot
$ sudo wget "${BASEURL}/centos-release-6-5.el6.centos.11.1.i686.rpm"
$ sudo rpm --root /tmp/sysroot --rebuilddb
$ sudo rpm --root /tmp/sysroot -i centos-release-6-5.el6.centos.11.1.i686.rpm
$ sudo yum --nogpgcheck --installroot=/tmp/sysroot groupinstall base
$ sudo sed -e 's/$releasever/6/g' -i /tmp/sysroot/etc/yum.repos.d/CentOS-Base.repo
{% endhighlight %}

You should now have a CentOS chroot within `/tmp/sysroot`.

Now we can package it up for Docker and use it.

{% highlight text %}
$ sudo tar -czf /tmp/sysroot.tgz -C /tmp/sysroot/ .
$ cat /tmp/sysroot.tgz | docker import - centos32
$ docker run -i -t centos32 /bin/bash
bash-4.1# 
{% endhighlight %}

I still prefer `debootstrap`.

  [1]: {{ site.url }}/post/running-docker-on-debian-i686
