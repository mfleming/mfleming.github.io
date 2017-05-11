---
title: Dear Btrfs, Where the fsck is my free space?
layout: post
comments: true
---

After booting my openSUSE Tumbleweed laptop this morning, I was
greeted with the following error messages in the console log:

{% highlight bash %}
[  459.834935] systemd-journald[706]: Failed to rotate /var/log/journal/dae9a2341321bbb6224dafc6561f8eb1/system.journal: No space left on device
[  459.841371] systemd-journald[706]: Failed to rotate /var/log/journal/dae9a2341321bbb6224dafc6561f8eb1/user-1000.journal: No space left on device
[  459.841551] systemd-journald[706]: Failed to write entry (24 items, 750 bytes), ignoring: Input/output error
{% endhighlight %}

It looked like the `/var/log` btrfs filesystem had run out of free
space, and that systemd could not create new files for its journal.

Thinking this was a run of the mill case of using up all my disk space
-- probably because of some huge log file -- I reached for
`df(1)`, and printed the filesystem statistics.

What I saw was totally unexpected:

{% highlight bash %}
$ df -h /var/log
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/system-root   40G   36G  3.8G  91% /var/log
{% endhighlight %}

Say what? If only 91% of the filesystem was in use, then why was I
hitting out of space errors?

Because, while there was plenty of available data space, I had
__exhausted the free *metadata* space__.

# Reclaim Metadata Space with snapper #

To fix this situation, so that I could create new files again, I had
to delete most of the btrfs snapshots for the `/` filesystem.

The default installation of openSUSE Tumbleweed creates a single btrfs
filesystem for the root filesystem (`/`), with btrfs subvolumes for
various other directories. Running out of free metadata space in a
subvolume (`/var/log`) means you've run out of space in the main
volume (`/`).

You can query the snapshots of the root filesystem using the
`snapper` command:

{% highlight bash %}
$ snapper -c root list
Type   | #  | Pre # | Date                     | User | Cleanup | Description           | Userdata     
-------+----+-------+--------------------------+------+---------+-----------------------+--------------
single | 0  |       |                          | root |         | current               |              
single | 1  |       | Thu Oct 22 09:51:54 2015 | root |         | first root filesystem |              
pre    | 11 |       | Tue Apr 25 16:24:54 2017 | root | number  | zypp(y2base)          | important=yes
pre    | 19 |       | Tue Apr 25 16:35:21 2017 | root | number  | zypp(zypper)          | important=yes
post   | 20 | 19    | Tue Apr 25 20:08:26 2017 | root | number  |                       | important=yes
pre    | 23 |       | Wed Apr 26 10:46:19 2017 | root | number  | zypp(zypper)          | important=yes
post   | 24 | 23    | Wed Apr 26 10:48:50 2017 | root | number  |                       | important=yes
pre    | 31 |       | Wed Apr 26 11:04:17 2017 | root | number  | zypp(zypper)          | important=no 
post   | 32 | 31    | Wed Apr 26 11:04:24 2017 | root | number  |                       | important=no 
pre    | 33 |       | Wed Apr 26 11:05:52 2017 | root | number  | zypp(zypper)          | important=no 
post   | 34 | 33    | Wed Apr 26 11:05:59 2017 | root | number  |                       | important=no 
pre    | 35 |       | Fri Apr 28 11:13:25 2017 | root | number  | zypp(zypper)          | important=no 
post   | 36 | 35    | Fri Apr 28 11:15:23 2017 | root | number  |                       | important=no 
pre    | 37 |       | Wed May  3 11:48:58 2017 | root | number  | zypp(zypper)          | important=yes
post   | 38 | 37    | Wed May  3 11:52:19 2017 | root | number  |                       | important=yes
pre    | 39 |       | Fri May  5 08:47:15 2017 | root | number  | zypp(zypper)          | important=yes
post   | 40 | 39    | Fri May  5 08:47:45 2017 | root | number  |                       | important=yes
pre    | 41 |       | Thu May 11 09:21:08 2017 | root | number  | zypp(zypper)          | important=yes
{% endhighlight %}

The pre and post snapshots are generated before and after
[YaST](https://en.opensuse.org/Portal:YaST) runs, respectively. As you
can see from the snapshot descriptions, YaST was executed on behalf of
[zypper](https://en.opensuse.org/Portal:Zypper); the openSUSE package
manager.

Every time I update packages with `zypper update`, two btrfs snapshots
are created. Because openSUSE Tumbleweed is a [rolling
release](https://en.opensuse.org/Portal:Tumbleweed) version of
openSUSE, this happens a lot.

To free up some metadata space I deleted every snapshot apart from the
current (snapshot `#0`) and initial snapshot (snapshot `#1`).

> __WARNING:__ The whole point of creating snapshots is to allow `zypper` to
rollback to previous states if you encounter issues. That will become
impossible if you follow the next step.

{% highlight bash %}
$ for snapshot in `echo $(seq 41 -1 31)` 24 23 20 19 11; do
	snapper -c root rm $snapshot
done
{% endhighlight %}

After that, it was possible to create files in `/var/lib` again, and
systemd was much happier.

If you're curious to know how I debugged this issue, and how you can
do the same for your btrfs filesystems, read on.

# Getting Serious with Btrfs Utilities #

One of the most surprising things about this whole ordeal was that
btrfs made zero attempts to tell me that I was hitting a metadata
`ENOSPC` issue; there was no error message of any kind in the console
log.

I think it's a natural instinct to pull out `df(1)` when you hit out
of space issues. But when it showed that I had plenty of free space on
the `/var/log` partition, I was stumped.

It turns out that btrfs has its own utilities for inspecting
partitions; all of them are described in the man page `btrfs(8)`. The
one I needed was `btrfs-filesystem(8)`, or `btrfs fi` for short.

{% highlight bash %}
$ btrfs fi df /var
Data, single: total=36.44GiB, used=32.65GiB
System, DUP: total=32.00MiB, used=16.00KiB
Metadata, DUP: total=1.75GiB, used=1.31GiB
GlobalReserve, single: total=448.00MiB, used=0.00B
{% endhighlight %}

Looking at the Metadata line shows that there's roughly 0.44GiB
of metadata space available. So why the error?

On every btrfs filesystem, some space is reserved so that the kernel
can always perform critical operations, like deleting files, even when
the filesystem is full. The GlobalReserve line above gives the size
of this emergency space.

Crucially, the GlobalReserve space is taken from the Metadata
pool, so you need to add the used Metadata size and the total
GlobalReserve size to calculate how much free metadata space you
*actually* have.

The `btrfs-filesystem(8)` [man
page](https://btrfs.wiki.kernel.org/index.php/Manpage/btrfs-filesystem)
even has this very helpful equation:

> In case the filesystem metadata are exhausted,
$$
\begin{align*}
GlobalReserve_{total} +
Metadata_{used} =
Metadata_{total}
\end{align*}
$$

After I deleted the snapshots I saw a much better picture of
filesystem statistics:
 
{% highlight bash %}
$ btrfs fi df /var
Data, single: total=36.44GiB, used=20.37GiB
System, DUP: total=32.00MiB, used=16.00KiB
Metadata, DUP: total=1.75GiB, used=661.02MiB
GlobalReserve, single: total=224.00MiB, used=0.00B
{% endhighlight %}

# A Permanent Fix #

The SUSE btrfs developers [area aware of this
issue](https://bugzilla.novell.com/show_bug.cgi?id=1036292). But that
bug is assigned to the kernel developers because they were interested
in thinking about how to make this scenario less painful at the kernel
level.

Which begs the question: Shouldn't `snapper` be deleting old snapshots
so that I don't run out of disk space and hit this issue again?

While `snapper` can delete snapshots once a [certain number have been
created](https://doc.opensuse.org/documentation/leap/reference/html/book.opensuse.reference/cha.snapper.html#sec.snapper.clean-up.number),
it does not provide a way to reduce that number if metadata space
starts to run out on the filesystem.

My only hope for a permanent fix today is to limit `snapper` to
keeping a single pre and a single post snapshot for important and
unimportant updates. That and pray they don't include too many files and
directories.

Again, note that the following suggestion is going to limit how far
you can rollback your system, should it stop working properly.

You can make the fix permanent by modifying
`/etc/snapper/configs/root` with the following settings:

{% highlight bash %}
# limit for number cleanup
NUMBER_MIN_AGE="1800"
NUMBER_LIMIT="2"
NUMBER_LIMIT_IMPORTANT="2"
{% endhighlight %}

Fingers crossed.
