---
title: "Does SMT still make sense?"
layout: post
---

Whatever machine you're reading this on, it's highly likely that not all
of the CPUs shown by the OS are actually physical processors. That's
because most modern processors use simultaneous multithreading (SMT) to
improve performance by executing tasks in parallel.

Intel's implementation of SMT is known as *hyperthreading*, and it was
originally introduced in 2002 as a way to improve the performance of
Pentium 4 and Xeon CPUs that didn't require increasing the clock
frequency. Most Intel CPUs supported the HyperThread Technology (HTT)
apart from the Core line of products until the Nehalem microarchitecture
which was introduced in 2008. Recently, Intel have announced that
they're [moving away from hyperthreading
again](https://arstechnica.com/gadgets/2018/07/leaked-benchmarks-show-intel-is-dropping-hyperthreading-from-i7-chips/)
with their Core product line.

AMD too have dabbled with SMT, and the diagram below shows how SMT works
in the Zen microarchitecture.

<center><img src="{{ site.baseurl }}/assets/HC28.AMD.Mike Clark.final-page-015.jpg"
  alt="AMD Zen SMT architecture diagram"/></center>

As the diagram above illustrates, some components are dedicated to each
thread and some are shared. Which pieces are shared? That depends on the
implementation and can potentially vary with the microarchitecture. But
it's usually some part of the execution unit. 

SMT threads usually come in pairs for x86 architecture, and those
threads contend with their siblings for access to shared processor
hardware. By using SMT you are effectively trying to exploit the natural
gaps in a thread's hardware use. Or as Pekka Enberg eloquently put it:

<center><blockquote class="twitter-tweet" data-conversation="none"><p
lang="en" dir="ltr">Simultaneous multithreading (SMT) is all about
keeping superscalar CPU units busy by converting thread-level
parallelism (TLP) to instruction-level parallelism (ILP). For
applications with high TLP and low ILP, SMT makes sense as a performance
optimization.</p>&mdash; Pekka Enberg (@penberg) <a
href="https://twitter.com/penberg/status/1191992460416827393?ref_src=twsrc%5Etfw">November
6, 2019</a></blockquote> <script async
src="https://platform.twitter.com/widgets.js"
charset="utf-8"></script></center>

There are both good and bad reasons to use SMT.

## Why is SMT good?

SMT implementations can be very efficient in terms of die size and power
consumption, at least when compared with fully duplicating processor
resources. 

With less than a 5% increase in die size, Intel claims that you can get
a [30% performance
boost](https://software.intel.com/en-us/articles/how-to-determine-the-effectiveness-of-hyper-threading-technology-with-an-application)
by using SMT for multithreaded workloads.

What you're likely to see in the real world depends very much on your
workload, and like all things performance-related, the only way to know
for sure is to benchmark it yourself.

Another reason to favour SMT is that it's now ubiquitous in modern x86
CPUs. So it's an easy, achievable way to increase performance. You just
need to make sure it's turned on in your BIOS configuration.

## Why is SMT bad?

One of the best and also worst things about SMT is that the operating
system doesn't make it obvious when SMT is enabled. Most of the time,
that's good because it's an unnecessary distraction. But when it comes
to things like capacity planning, or tuning your system for real-time
workloads, you really need to know if SMT is enabled.

Take assigning physical CPUs to virtual machines, for example. If you're
not aware that SMT is turned on, it's easy to think that doubling the
number of CPUs will double the performance, but that's almost certainly
not going to be the case if those CPUs are actually SMT siblings because
they will be contending for processor resources. 

Modern x86 processors come with so many cores (the new [AMD Rome
CPUs](https://www.amd.com/en/processors/epyc-7002-series) processors
have 64 and top-line Intel Core i9 CPUs now have 18) that there's plenty
of performance to be had without enabling SMT.

But perhaps the biggest reason against SMT, and this is definitely the
Zeitgeist, is the string of security vulnerabilities that have been
disclosed over the last year or so, including
[L1TF](https://www.kernel.org/doc/html/latest/admin-guide/hw-vuln/l1tf.html)
and
[MDS](https://www.kernel.org/doc/html/latest/admin-guide/hw-vuln/mds.html).

OpenBSD was the first operating system to [recommend disabling SMT
altogether](https://marc.info/?l=openbsd-tech&m=153504937925732&w=2) in
August 2018 because they feared that more vulnerabilities would be
discovered in this area. They were right.

Greg Kroah Hartman gave a shout out to OpenBSD's forward thinking in his
Open Source Summit [talk last
month](https://www.theregister.co.uk/2019/10/29/intel_disable_hyper_threading_linux_kernel_maintainer/)
where he said that he had "huge respect" for the project because they
made a tough call and chose security over performance.

## How to disable SMT

Many users are also now considering disabling SMT all together. Whether
or not you're willing to do that depends on your individual
circumstances.

But if you are thinking about it, you'll need to run benchmarks to
understand the performance impact. Which means you'll need a handy way
to run with SMT on and off.

This is usually done in the BIOS setup but if you can't access it -- and
if you're running Linux -- you can disable it by writing "off" to the
`/sys/devices/system/cpu/smt/control` file, e.g.

{% highlight bash %}
$ nproc
4
$ echo off > /sys/devices/system/cpu/smt/control
$ nproc
2
{% endhighlight %}

To re-enable SMT you just need to write "on" to the same file.

So does SMT still make sense? There's no reason to think that we've seen
the last of chip-level security vulns, so if you're at all concerned
about security then the safest best is to run with SMT disabled. If
you're concerned about losing performance, you need to get to running
those benchmarks.
