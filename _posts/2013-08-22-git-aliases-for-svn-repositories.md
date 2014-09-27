---
layout: post
title: Git Aliases for SVN Repositories
date: 2013-08-22 14:27:51.000000000 -07:00
status: published
type: post
---
I use Git often but I've also encountered SVN repositories that I would
like to use. Git-SVN provides the low-level integration with SVN but the
steps required to set it up can be a hassle. Because of this, I've
created some Git aliases to assist in speeding things up.

{% highlight ini %}
# ~/.gitconfig
[alias]
    rebase-svn = svn rebase -q
    commit-svn = svn dcommit -q
    clone-svn  = svn clone -q --prefix=svn/ -s
    track-svn  = branch svn-sync svn/trunk
    setup-svn  = !sh -c 'git clone-svn \"$1\" \"$PWD\" && git track-svn' -
{% endhighlight %}

Now I can initialize a currently existing Git repository to track an SVN
repository as well.

{% highlight bash %}
# Create a Git repository for testing.
$ git init test-git-repository && cd test-git-repository
Initialized empty Git repository in /home/user/test-git-repository/.git/
$ git setup-svn svn+ssh:///hostname.domain.tld/path/to/repository/
$ git branch
* master
  svn-sync
# When you're ready to send your commits to master upstream...
$ git checkout svn-sync
$ git rebase-svn master
$ git rebase-svn svn/trunk
$ git commit-svn
Committing to file:///home/user/test-svn-repository/trunk ...
...
{% endhighlight %}

The "svn-sync" branch will track the remote SVN repository. You get all of
the benefits of Git's decentralized model -- like a full copy of the remote
repository and history -- while still retaining the ability to commit to
the upstream SVN repository.
