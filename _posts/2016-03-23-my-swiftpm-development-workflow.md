---
layout: post
title: My Swift-PM Development Workflow
date: 2016-03-23 01:28
categories: Swift
---

Recently I've started contributing to [Swift Package Manager](https://github.com/apple/swift-package-manager/). Contributing to Open Source projects is really an awesome learning experience. This post is about my workflow for contributing to Swift-PM.

## Setup
- Macbook Pro
- Ubuntu-15.10, 64-bit, Server edition running in [VirtualBox](https://www.virtualbox.org)

## Password less SSH
By using authorization keys, we can SSH into Ubuntu VM with out entering password everytime . By default, public key for SSH is stored in `~/.ssh/id_rsa.pub`. If you don't have the file, creating it is very simple:

{% highlight bash %}
$ ssh-keygen -t rsa   #Generate new SSH key
{% endhighlight %}

The above command will ask for file path and password. These are optional, just press enter to accept defaults.

Copy your generated SSH key from `~/.ssh/id_rsa.pub` (unless another path is provided).

Now, from OS X, SSH into Ubuntu with username/password and do the following:
{% highlight bash %}
$ # On Ubuntu
$ echo "COPIED_SSH_KEY" >> ~/.ssh/authorized_keys
{% endhighlight %}

Logout and from the next time onwards, we can login without password.

## Workflow - Development
All of my development happens on OS X and I use Ubuntu only for running final tests. Before I start to work, I follow the usual process of updating your local `master` branch with server and creating a new branch out of it as follows:

{% highlight bash %}
# On Mac
$ git checkout master
$ git pull origin master  #keep up-to-date with server
$ git checkout -b "my-feature-branch"  #create a new branch to work
# Hack... Hack... Hack...
# Time to test
# rsync files to Ubuntu VM
# ssh to VM and run the tests
{% endhighlight %}

Because all the changes are on OS X, code has to be copied to Ubuntu box to test it.

Options to copy the code:

1. **Use shared folders in VirtualBox:** I couldn't get this to work, all shared folders are mounted by root user and I have no write permission. Also, tests can't be run parallelly on both OS X and Ubuntu.
2. **Push the changes to GitHub from Mac machine and pull them back on Ubuntu:** Not a big fan of this approach.

## `rsync` to the rescue

`rsync` is an awesome file synchronization utility that can work over SSH. Best part is, it only copies the files that are only changed since last copy.

The below command copies all the Swift-PM sources from OS X to Ubuntu VM.

{% highlight bash %}
#On Mac
$ rsync --verbose \
        --update  \
        --archive \
        --exclude=".build"  \
        --exclude=".git"    \
        HOST_SPM_DIRECTORY 
        VM_USERNAME@VM_IP:VM_SPM_DESTINATION
{% endhighlight %}

## Workflow - Testing

**Testing on OS X:**

1. Run `Utilities/bootstrap && .build/debug/swift-test`

**Testing on Ubuntu VM:**

1. `rsync` code
2. SSH into Ubuntu
3. `cd spm/path`
2. Run `Utilities/bootstrap && .build/debug/swift-test`

Below script automates the above steps and runs the tests with a single command `test-spm`

{% gist bhargavg/cb0a59fb523302039e43 %}

The above script works for both Ubuntu and OS X. Download the gist file as `script.sh` and do the following:

{%  highlight bash %}
# On Mac or Ubuntu
$ source script.sh
$ export HOST_IP="ip.address.of.OSX"
$ export HOST_USERNAME="output_of_whoami_on_OSX"
$ export HOST_SPM_DIRECTORY="path/to/spm/on/OSX"
$ export VM_SPM_PARENT="where/you/want/spm/to/be/on/Ubuntu"
$ test-spm   # Start testing
{% endhighlight %}

Best part is, you can use `tmux` or multiple terminal tabs to run the tests in parallel on both Ubuntu VM and OS X.

> Note: The `source` and `export` statements could be copied to `~/.bash_profile` or `~/.zsh_profile` (depending on your shell) so that we don't have to set it everytime.

Once all the tests pass, I'll `rebase` the changes onto `upstream/master`, `squash` unwanted commits and raise a PR.
