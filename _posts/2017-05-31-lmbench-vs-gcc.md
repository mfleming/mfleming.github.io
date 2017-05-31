---
title: LMBench versus GCC Optimisations
layout: post
comments: true
---

A quality benchmark is authoritative and trustworthy, and when you're
using one it's a bit like playing the [card game
Snap](https://en.wikipedia.org/wiki/Slapjack#Snap): the rules are
easy, and when the game is over it's obvious who won.

But a poor benchmark makes performance work more like trying to solve
a twisted version of the [Knights and
Knaves](https://en.wikipedia.org/wiki/Knights_and_Knaves) riddle where
you're not sure if the answers you're getting are truths or lies, no
one ever wins, and you only stop playing because you're exhausted.

[LMBench](http://www.bitmover.com/lmbench/) definitely has that riddle vibe.

I just don't trust the test results that it spits out because I've run
into too many dead ends when investigating performance issues that
turned out to be false positives. And if there's one thing that you
need to be sure of when measuring performance, it's the [accuracy of
your
results](https://www.dynatrace.com/blog/why-averages-suck-and-percentiles-are-great/).

So I was less than convinced when I recently saw that the *int64-mul*
subtest of the LMBench *ops* microbenchmark was taking between 10% and
20% longer to run with a new enterprise kernel.

With my suspicions suitably heightened, I started reading the source
code to understand exactly what the test was doing.

The *int64-mul* subtest tests the CPU speed of 64-bit integer
multiplication. Here's an edited version:

{% gist mfleming/c55bc466d0d359f6ef5799eb3c0b1446 %}

Seeing the `register` keyword always sets alarm bells ringing for me.
Not because it has no purpose -- you can use it to [disallow using the
unary address-of operator on a
variable](https://gcc.gnu.org/ml/gcc/2010-05/msg00098.html), which
lets the compiler optimise accesses to that variable -- but because it
usually indicates that the benchmark has been written with a specific
compiler implementation, or version, in mind. LMBench was released in
1996 which would have made [GCC 2.7 the current
version](https://gcc.gnu.org/releases.html).

Using the `register` keyword may have helped old compilers optimise
access to variables by allocating registers for them, but modern
compilers ignore `register` when making register allocation decisions.

Before doing anything else, I wanted to verify that the compiler was
emitting those 64-bit multiplication operations on lines 17-21 above. 

{% highlight raw %}
00000000004004cb <do_int64_mul>:
  4004cb:       89 f2                   mov    %esi,%edx
  4004cd:       8d 46 06                lea    0x6(%rsi),%eax
  4004d0:       48 c1 e0 20             shl    $0x20,%rax
  4004d4:       48 8d 84 02 2c 92 00    lea    0x922c(%rdx,%rax,1),%rax
  4004db:       00 
  4004dc:       83 ef 01                sub    $0x1,%edi
  4004df:       83 ff ff                cmp    $0xffffffff,%edi
  4004e2:       75 f8                   jne    4004dc <do_int64_mul+0x11>
  4004e4:       89 c7                   mov    %eax,%edi
  4004e6:       e8 cb ff ff ff          callq  4004b6 <use_int>
  4004eb:       f3 c3                   repz retq 
{% endhighlight %}

Nope. There's a complete lack of 64-bit multiplication anywhere in
there. As far as the compiler is concerned, the following C code is
equivalent to LMBench's `do_int64_mul()`:

{% gist mfleming/1582c252477364bca0ea121615f26131 %}

Which makes the test useless because **GCC optimised it away**.

# Why did GCC optimise out the test? #

GCC could tell exactly how many times it needed to add all of those
64-bit constants together and used techniques like [Constant folding
and propagation](https://en.wikipedia.org/wiki/Constant_folding) to
calculate the end value at compile time instead of runtime.

> While investigating this issue I discovered that GCC didn't throw
> away the useless loop on lines 8-9 because LMBench uses the -O
> switch which doesn't include the necessary optimisation flag. Here's
> the full [list of
> optimisations](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)
> and which level they are enabled for.

This is the problem with microbenchmarks that assume a specific
toolchain version or implementation -- upgrading the toolchain can
break them without you realising. Instead of writing the inner loops
in C (the authors [wanted it to be
portable](http://www.bitmover.com/lmbench/why_lmbench.html)), inline
assembly would have prevented the compiler from eliminating them.

Tests like *int64-mul* are so low-level that I've heard them referred
to as *nanobenchmarks*; they are notoriously easy to misuse and
misunderstand. Here's Aleksy Shipilëv, infamous JVM performance
expert, showing how to use them with
[JMH](http://openjdk.java.net/projects/code-tools/jmh/), a benchmark
harness:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Meanwhile, this is yet another example how you should approach nanobenchmarks (and <a href="https://twitter.com/hashtag/JMH?src=hash">#JMH</a> makes it convenient enough): <a href="http://t.co/vw0jVt8x0d">http://t.co/vw0jVt8x0d</a></p>&mdash; Aleksey Shipilëv (@shipilev) <a href="https://twitter.com/shipilev/status/622135063531155457">July 17, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

# Is it time to retire LMBench? #

As much as I distrust LMBench, I actually plan to keep using it. Why?
Because it has some other subtests that are useful, like the fork()
microbenchmark test, which detected the overhead of the
`vm_ops->map_pages()` API [when it was
introduced](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8c6e50b0290c4c708a3e6462729e1e9151a9a7df).

But the CPU ops subtest? No, that nanobenchmark definitely needs to go
in the trash.
