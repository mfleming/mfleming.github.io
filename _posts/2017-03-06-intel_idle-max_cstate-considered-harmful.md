---
title: intel_idle.max_cstate=0 Considered Harmful
layout: post
comments: true
---

The other week I had a rather illuminating discussion with an Intel
engineer about trying to get the most performance out of a machine by
disabling C-states.

### Why would you want to disable C-states? ###

C-states help the [processor to conserve power when
idle](https://software.intel.com/en-us/articles/power-management-states-p-states-c-states-and-package-c-states#_Toc383778910),
but entering and exiting these lower powered states incurs latency.
The higher a C-state you transition to the more power you save, but
the more time it takes getting in and out.

Since an idle processor might be in one of these C-states when new
work arrives, the conventional wisdom is that you should disable
C-states if you want to reduce latency for your workload.

The Internet is awash with forum posts that explain how to do this by
specifying the kernel parameter `intel_idle.max_cstate=0`. I had a
customer doing exactly that.

Now, the engineer claimed that preventing idle processors from using
C-states may actually *hurt* performance rather than help it, and the
reasoning goes like this...

If you run your processor core with hyper threading enabled then
thread siblings [share some pipeline
resources](https://en.wikipedia.org/wiki/Hyper-threading) -- they
cannot run concurrently if they're trying to use a shared piece of
hardware. Allowing one thread to enter an idle state (C1 or higher)
provides an obvious opportunity for the sibling thread to use those
resources. If you force all threads to run at C0, these natural
scheduling points never occur and the processor core has to work that
much harder to schedule threads.

### Less latency, more performance? ###

I wanted to test this theory. So I grabbed a 2 socket NUMA machine
with 12-core Intel Xeon E5 CPUs (hyper threading enabled) and the
results of running the scheduler benchmark, hackbench, are below
(lower time on the y-axis is better).

![hackbench]({{ site.baseurl }}/assets/hackbench.png)

Allowing the processors to go to C1 improved the score by `11.14%`. I
ran hackbench with 40 tasks so the machine, with its 48 CPUs, was
never at 100% utilisation, but still, the results are not to be
sniffed at.

I would expect the difference to vary with microarchitecture and
processor technology, but clearly there's some merit to this claim.

Check your workloads.
