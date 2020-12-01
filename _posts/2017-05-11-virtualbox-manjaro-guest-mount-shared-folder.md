---
layout: post
title: "Mounting Virtualbox Shared Folder in Manjaro Guest"
description: "Everytime I setup a Linux vm in Virtualbox and attempt to get shared folders working, I always run into issues. I decided to create a blog post to save my self the trouble of Googling so that I have the information in one place. In the various distros I have encountered issues with, Manjaro being the most recent, the issues have seemed to have been addressed by three things."
tags: [manjaro, virtualbox]
---

Everytime I setup a Linux vm in Virtualbox and attempt to get shared
folders working, I always run into issues. I decided to create a blog
post to save my self the trouble of Googling so that I have the
information in one place. In the various distros I have encountered
issues with, Manjaro being the most recent, the issues have seemed to
have been addressed by three things.


## Groups

In all cases it has been important that the user who will be using the
shared directory be a part of the `vboxsf` group. This is achieved
with:

{% highlight bash %}

sudo usermod -a UserToAddGroupTo -G vboxsf

{% endhighlight %}

I have also found that it is essential to reboot after this change.

## Mounting

I also seem to encounter issues with auto-mounting on various
distros. To mount a virtualbox shared folder the following seems to
work:

{% highlight bash %}

mkdir /home/username/mountpoint-name

sudo mount.vboxsf -t vboxsf -o gid=1000,uid=1000,rw SharedFolderNameInHost /home/username/mountpoint-name

{% endhighlight %}

This takes care of mounting for a single session but does not address
auto-mounting.

## Auto-Mounting

Auto-mounting has worked well in Debian based distros like Kali Linux
and Ubuntu but Arch Linux and Manjaro have given me issues. The way I
have addressed this in the past has been to create an fstab
entry. This seems unfortunately necessary but has seemed to address
the auto-mount problem.


## Conclusion

Virtualbox shared folders have different levels of annoyance depending
on the distro in use. If any readers have any tips they would like to
add, please leave me a comment and I will update my post.
