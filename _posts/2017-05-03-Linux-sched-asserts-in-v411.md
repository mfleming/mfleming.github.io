---
title: New Linux scheduler runqueue clock asserts in v4.11
layout: post
comments: true
---

![clock]({{ site.baseurl }}/assets/cuckoo-clock-397360.jpg){: .center-image}

Linux v4.11 was released this week and it includes my patches that add
new diagnostic checks to the scheduler. There was quite a bit of fallout
when they were merged, and even [Linus triggered one of the
asserts](https://marc.info/?l=linux-kernel&m=148779611401037).

Dramatic start or not, the new checks are useful for detecting a
long-standing problem in the kernel; missing updates to the 
scheduler's runqueue clocks.

## Runqueue clocks ##

Fundamentally the Linux kernel scheduler cares about three things:
what to run, where to run and when. That is to say, it cares about
tasks (what), CPUs (where) and time (when).

To know when to run tasks, the scheduler needs an accurate view of
time. It needs to know:

 - How long a task has been running and if it needs preempting
 - Which task ran least recently and should be scheduled next
 - If it's time to do housekeeping work like load balancing tasks
 - Average idle duration of CPUs for housekeeping cost estimation

Continuously tracking the current time would be prohibitively
expensive, so every scheduler runqueue maintains its own software
clock that is periodically synchronised with hardware. Basically, the
runqueue clock holds a copy of the value read from a hardware clock.
It's the timestamp of when the runqueue clock was updated.

Exactly when the runqueue clock gets updated varies from code path to
code path. There are many calls to `update_rq_clock()` (I count 35 in
Linus' tree), and it's extremely easy to forget to add one. And
therein lies the problem...

## Missing clock updates ##

It turned out that there were a bunch of places inside the kernel that
didn't correctly update the runqueue clocks before doing important
operations. This went unnoticed because nothing *checked* that the
clocks had been updated.

One example was where updating a task's load average when moving it
between cgroups was using an [incorrect view of
time](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1b1d62254df0fe42a711eb71948f915918987790),
another was how the deadline scheduler read the potential stale clock
when [calculating a task's deadline after migrating to a new
CPU](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=dcc3b5ffe1b32771c9a22e2c916fb94c4fcf5b79).

Missing updates don't necessarily cause crashes, and which is why not
checking for updates is so dangerous; things just silently
keep working in spite of the logic errors.

## The efi.flags example ##

This isn't the first example of implicit API agreements going
unverified inside the kernel. I accidentally wrote one. When
the `efi.flags` code [was
introduced](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3e909599215456928e6b42a04f11c2517881570b),
it was intended to be a way to query the features of a machine's EFI
firmware.

![git commit]({{site.baseurl}}/assets/git-commit-efi-flags.jpg){: .center-image}

I audited all users of `efi.flags` when I added the wrapper to ensure
that flags were being set correctly. But as new users were added to
the kernel bugs were introduced where flags were unset even though they
[were
required](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9479c7cebfb568f8b8b424be7f1cac120e9eea95).

And this happened despite me reviewing all changes to the EFI
subsystem. It's the kind of issue that is best detected by the
internals of the API at runtime. With assertions.

## Assertions are contracts ##

As with any large open source project, patches get applied, code
changes and while everything might have worked at one point in time,
that is unlikely to be true forever.

Asserts, or something similar, should have been introduced at the
start for both examples above. Asserts keep your code correct even as
it evolves. Baking the expected state into your API checks makes
this contract between API provider and user concrete. 

And remember The Golden Rule of using assertions in the Linux kernel:
[Don't use
BUG_ON](https://marc.info/?l=linux-kernel&m=148752591510031&w=2).
