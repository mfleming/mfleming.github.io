---
title: "isolcpus is deprecated, kinda"
layout: post
excerpt: "For most uses, isolcpus should be considered deprecated and cset is the recommended option."
---

A problem that a lot of sysadmins and developers have is, how do you run
a single task on a CPU without it being interrupted? It's a common
scenario for real-time and virtualised workloads where any interruption
to your task could cause unacceptable latency.

For example, let's say you've got a virtual machine running with 4
vCPUs, and you want to make sure those vCPU tasks don't get preempted by
other tasks since that would introduce delays into your audio
transcoding app.

Running each of those vCPU tasks on its own host CPU seems like the way
to go. All you need to do is choose 4 host CPUs and make sure no other
tasks run on them.

How do you do that?

I've seen many people turn to the kernel's `isolcpus` for this. This
kernel command-line option allows you to run tasks on CPUs without
interruption from a) other tasks and b) kernel threads.

But `isolcpus` is almost never the thing you want and you should
absolutely not use it apart from one specific case that I'll get to at
the end of this article.

So what's the problem with isolcpus?

# 1. Tasks are not load balanced on isolated CPUs

When you isolate CPUs with `isolcpus` you prevent all kernel tasks from
running there and, crucially, it prevents the Linux scheduler load
balancer from placing tasks on those CPUs too. And the only way to get
tasks onto the list of isolated CPUs is with `taskset`. They are
effectively invisible to the scheduler.

Continuing with our audio transcoding app running on 4-vCPUs example
above, let's say you've booted with the following kernel command-line:
`isolcpus=1-4` and you use `taskset` to place your four vCPU tasks on to
those isolated CPUs like so: `taskset -c 1-4 -p <vCPU task pid>`

The thing that always catches people out is that it's easy to end up
with all of your vCPU tasks running **on the same CPU**!

{% highlight bash %}
$ ps -aLo comm,psr | grep qemu
qemu-system-x86 1
qemu-system-x86 1
qemu-system-x86 1
qemu-system-x86 1
{% endhighlight %}

Why? Well because `isolcpus` disabled the scheduler load balancer for
CPUs 1-4 which means the kernel will not balance those tasks equally
among all the CPUs in the affinity mask. You can work around this by
manually placing each task onto a single CPU by adjusting its affinity.

# 2. The list of isolated CPUs is static

A second problem with `isolcpus` is that the list of CPUs is configured
statically at boot time. Once you've booted, you're out of luck if you
want to add or remove CPUs from the isolated list. The only way to
change it is by rebooting with a different `isolcpus` value.

# cset to the rescue

My recommended way to [run tasks on CPUs without
interruption](https://documentation.suse.com/sle-rt/15-SP1/html/SLE-RT-all/book-slert-shielding.html)
by isolating them from the rest of the system with the cgroups subsystem
via the `cset shield` command, e.g.

{% highlight bash %}
$ cset shield --cpu 1-4 --kthread=on
cset: --> shielding modified with:
cset: kthread shield activated, moving 34 tasks into system cpuset...
[==================================================]%
cset: **> 34 tasks are not movable, impossible to move
cset: "system" cpuset of CPUSPEC(0,3) with 1694 tasks running
cset: "user" cpuset of CPUSPEC(1-2) with 0 tasks running
$ cset shield --shield --pid <vCPU task pid 1>,<vCPU task pid 2>,<vCPU task pid 3>,<vCPU task pid 4>
cset: --> shielding following pidspec: 17063,17064,17065,17066
cset: done
{% endhighlight %}

With `cset` you can update and modify the list of CPUs included in the
cgroup dynamically at runtime. It is a much more flexible solution for
most users.

# Sometimes you really do want isolcpus

OK, I admit there are times when you really do want to use `isolcpus`.
For those scenarios when you really cannot afford to have your tasks
interrupted, not even by the scheduler tick which fires once a second,
you should turn to `isolcpus` and manually spread tasks over the CPU
list with `taskset`.

But for most uses, `cset shield` is by far the best option that's least
likely to catch you by surprise.
