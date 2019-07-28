---
layout: post
comments: true
title: Adding timestamps to bash scripts with ts
---

I don't know when I started writing a lot of bash scripts. It just
seemed to happen over the last few years, possibly because bash is
pretty much universally available, even on newly installed Linux
systems.

Despite its benefits, one of the things I really hate about bash is
logging. Rolling my own timestamps can be a real PITA, but they're so
useful that I can't live without them (optimising software performance
for a living gives you a rather unhealthy obsession with how long things
take). Every script I write ends up using a different timestamp format
because I just can't seem to remember the `date` command I used last.

At least, that was the old me. The new me has discovered the perfect
tool: `ts` from the [moreutils](https://joeyh.name/code/moreutils/)
package.

`ts` prepends a timestamp to each line it receives on stdin. Adding a
timestamp in your log messages is a simple as:

{% highlight bash %}
$ echo bar | ts
Jul 28 22:27:51 bar
{% endhighlight %}

You can also specify a `strftime(3)` compatible format:

{% highlight bash %}
$ echo bar | ts "[%F %H:%M:%S]"
[2019-07-28 22:34:48] bar
{% endhighlight %}

But wait, there's more! If a simple way to print timestamps wasn't
enough, `ts` can also parse *existing* timestamps in the input line (by
feeding your `ts`-tagged logs back into `ts`) and preprend an additional
timestamp with cumulative and relative times between consecutive lines.

This is fantastic for answering two questions:

1. When was a log message printed relative to the start of the program? (`-s`)

2. When was a log message printed relative to the previous line? (`-i`)

The `-s` option tells you how long it took to reach a certain point of
your bash script. And the `-i` option helps you which parts of your
script are taking the most time.

{% highlight bash %}
$ cat sleeper.sh && ./sleeper.sh 
#!/bin/bash
FMT="[%H:%M:%S]"
for i in 1 2 3; do
	echo "Step $i"
	sleep $((i*10))
done | ts $FMT | ts -s $FMT | ts -i "[+%s]" | awk '
BEGIN { print "Timestamp | Runtime | Delta";
	print "---------------------------" }

{
	# ts(1) only prepends. Rearrange the timestamps.

	printf "%s %s %s ", $3, $2, $1;

	for (i=4; i <= NF; i++) {
		printf "%s ", $i;
	}

	printf "\n";
}'
Timestamp | Runtime | Delta
---------------------------
[23:15:22] [00:00:00] [+0] Step 1 
[23:15:32] [00:00:10] [+10] Step 2 
[23:15:52] [00:00:30] [+20] Step 3 
$
{% endhighlight %}

Beat that, hand-rolled `date` timestamps.
