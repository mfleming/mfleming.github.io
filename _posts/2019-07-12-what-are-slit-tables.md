---
title: "What are ACPI SLIT tables and why are they filled with lies?"
layout: post
comments: true
---

Let's say you've got a computer with a single processor and you want more
computational power. A good option is to add more processors since they can
mostly use the same technologies already present in your computer. Boom. Now
you've got a symmetric multiprocessing (SMP) machine.

But pretty quickly you're going to run into one of the drawbacks of SMP
machines with a single main memory: bus contention. When data requests cannot
be serviced from a processor's cache, you get multiple processors trying to
access memory via a single bus. The more processors you have, the worse the
contention gets.

To fix this, you might decide to attach separate memory to each processor and
call it *local memory*. Memory attached to remote processors becomes *remote
memory*. Now each processor can access its own local memory using a separate
memory bus.  This is what many manufacturers started doing in the early 1990's,
and they called it [Non-uniform Memory
Access](https://en.wikipedia.org/wiki/Non-uniform_memory_access) (NUMA), and
named each group of processor, local memory, and I/O buses a *NUMA node*.

NUMA systems improve the performance of most multiprocessor workloads, but it's
rare for any workload to be perfectly confined to a single NUMA node. All kinds
of things can happen that result in remote memory accesses, such as a task
being migrated to a new NUMA node or running out of free local memory.

This might sound like the pre-1990 memory bus contention problem all over
again, but the situation is actually worse now. Accessing remote memory takes
much longer than accessing local memory, and not all remote memory has the
same access latency. Depending on how the memory architecture is configured,
NUMA nodes can be multiple *hops* away with each hop adding more latency.

So when a task exhausts all of local memory and the Linux kernel needs to grab
free block from a remote node it has a decision to make: which remote node has
enough free memory **and** the lowest access latency?

To figure that out, the kernel needs to know the access latency between any two
NUMA nodes. And that's something the firmware describes with the ACPI System
Locality Distance Information Table (SLIT).

# What are SLIT tables?

System firmware provides ACPI SLIT tables (described in the section 5.2.17 of
the [ACPI 6.1
specification](https://uefi.org/sites/default/files/resources/ACPI_6_1.pdf))
that describe the relative differences in access latency between NUMA nodes.
It's basically a matrix that Linux reads on boot to build a map of NUMA memory
latencies, and you can view it multiple ways: with `numactl`, the
`node/nodeX/distance` sysfs file, or by dumping the ACPI tables directly with
`acpidump`.

Here is the data from my NUMA test machine. It only has two NUMA nodes so it
doesn't give a good sense of how the node distances can vary, but at least
we've got much less data to read.

{% highlight bash %}
$ numactl -H
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 24 25 26 27 28 29 30 31 32 33 34 35
node 0 size: 31796 MB
node 0 free: 30999 MB
node 1 cpus: 12 13 14 15 16 17 18 19 20 21 22 23 36 37 38 39 40 41 42 43 44 45 46 47
node 1 size: 32248 MB
node 1 free: 31946 MB
node distances:
node   0   1 
  0:  10  21 
  1:  21  10 
{% endhighlight %}

{% highlight bash %}
$ cat /sys/devices/system/node/node*/distance
10 21
21 10
{% endhighlight %}

{% highlight bash %}
$ acpidump > acpidata.dat
$ acpixtract -sSLIT acpidata.dat 

Intel ACPI Component Architecture
ACPI Binary Table Extraction Utility version 20180105
Copyright (c) 2000 - 2018 Intel Corporation

  SLIT -      48 bytes written (0x00000030) - slit.dat
$ iasl -d slit.dat 

Intel ACPI Component Architecture
ASL+ Optimizing Compiler/Disassembler version 20180105
Copyright (c) 2000 - 2018 Intel Corporation

Input file slit.dat, Length 0x30 (48) bytes
ACPI: SLIT 0x0000000000000000 000030 (v01 SUPERM SMCI--MB 00000001 INTL 20091013)
Acpi Data Table [SLIT] decoded
Formatted output:  slit.dsl - 1278 bytes

$ cat slit.dsl
/*
 * Intel ACPI Component Architecture
 * AML/ASL+ Disassembler version 20180105 (64-bit version)
 * Copyright (c) 2000 - 2018 Intel Corporation
 * 
 * Disassembly of slit.dat, Thu Jul 11 11:53:37 2019
 *
 * ACPI Data Table [SLIT]
 *
 * Format: [HexOffset DecimalOffset ByteLength]  FieldName : FieldValue
 */

[000h 0000   4]                    Signature : "SLIT"    [System Locality Information Table]
[004h 0004   4]                 Table Length : 00000030
[008h 0008   1]                     Revision : 01
[009h 0009   1]                     Checksum : DE
[00Ah 0010   6]                       Oem ID : "SUPERM"
[010h 0016   8]                 Oem Table ID : "SMCI--MB"
[018h 0024   4]                 Oem Revision : 00000001
[01Ch 0028   4]              Asl Compiler ID : "INTL"
[020h 0032   4]        Asl Compiler Revision : 20091013

[024h 0036   8]                   Localities : 0000000000000002
[02Ch 0044   2]                 Locality   0 : 0A 15
[02Eh 0046   2]                 Locality   1 : 15 0A

Raw Table Data: Length 48 (0x30)

  0000: 53 4C 49 54 30 00 00 00 01 DE 53 55 50 45 52 4D  // SLIT0.....SUPERM
  0010: 53 4D 43 49 2D 2D 4D 42 01 00 00 00 49 4E 54 4C  // SMCI--MB....INTL
  0020: 13 10 09 20 02 00 00 00 00 00 00 00 0A 15 15 0A  // ... ............

{% endhighlight %}

The distances (memory latency) between a node and itself is normalised to 10,
and every other distance is scaled relative to that 10 base value. So in the
above example, the distance between NUMA node 0 and NUMA node 1 is 2.1 and
the table shows a value of 21. In other words, if NUMA node 0 accesses memory
on NUMA node 1 or vice versa, access latency will be 2.1x more than for local
memory.

At least, that's what the ACPI tables claim.

# How accurate are the SLIT table values?

It's a perennial problem with ACPI tables that the values stored are not always
accurate, or even necessarily true. Suspecting that might be the case with SLIT
tables, I decided to run [Intel's Memory Latency
Checker](https://software.intel.com/en-us/articles/intelr-memory-latency-checker)
tool to measure the actual main memory read latency between NUMA nodes on my
test machine.

Measuring memory latency is notoriously difficult on modern machines because of
hardware prefetchers. Fortunately, MLC disables prefetchers using the Linux
`msr` module but that also means you need to run it as root.

{% highlight bash %}
$ ./Linux/mlc --latency_matrix
Intel(R) Memory Latency Checker - v3.7
Command line parameters: --latency_matrix 

Using buffer size of 200.000MiB
Measuring idle latencies (in ns)...
		Numa node
Numa node	     0	     1	
       0	  78.5	 119.4	
       1	 120.8	  76.6	
{% endhighlight %}

A bit of quick maths shows that these latency measurements do not match with
the SLIT values. In fact, the relative distance between node 0 and 1 is closer to 1.5.

# OK, but who cares?

Remember when I said that NUMA node distances were used by the kernel? One of
the places they're used is figuring out whether to load balance tasks between nodes. Moving
tasks across NUMA nodes is costly, so the kernel tries to avoid it whenever it can.

How costly is too costly? The current [magic
value](https://github.com/torvalds/linux/blob/master/include/linux/topology.h#L60)
used inside Linux kernel is 30 -- if the NUMA node distance between two
nodes is more than 30, the Linux kernel scheduler will try not to migrate tasks
between them.

Of course, if the ACPI tables are *wrong*, and claim a distance of 30 or more but in reality it's less, you're unnecessarily impacting
performance because you'll likely see one extremely busy NUMA node while over
nodes sit idle. And the scheduler will refuse to migrate tasks to the idle
nodes until the active load balancer kicks in.

[This
patch](https://lore.kernel.org/lkml/20190605155922.17153-1-matt@codeblueprint.co.uk/)
fixes this exact bug for AMD EYPC machines which report a distance of 32
between some nodes (I measured the memory latency with `mlc` and it's
definitely less than 3.2x). With the patch applied you get a nice 20-30%
improvement to CPU-intensive benchmarks because the scheduler balances tasks
across the entire system.

Ugh. Firmware.
