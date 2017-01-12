---
layout: post
title: Optimising The Common Case In The Real World
---

There are plenty of articles that talk about "optimising for the
common case", but nothing drives that rule home like encountering it
in the real world, as I discovered recently while having dinner with
my friends, Jim and Anna.

Jim does most of the cooking in their household (he's mad about
cooking), so Christmas and birthdays usually result in a few cooking
gadgets being bought as presents, all of which need to find new homes
in the kitchen. Since Anna is the better organiser, this job always
falls to her.

After the kitchen has been rearranged, she always gives Jim a
walk through of where things are now located: "See? The new blender is
next to the fruit bowl so we can have smoothies every morning for
breakfast, and I put the [Progressive International Green Asparagus
Peeler](https://www.amazon.com/dp/B00296C62I/?tag=kitchn-20) on the
hook next to pots so you'll always know where it is."

But the next time Jim tries to cook a meal he invariably discovers
that one of the tools he uses the most is now stored at the back of
the cupboard, or on top of it, or has been moved out of the kitchen
altogether and now lives in a box under the stairs.

Jim's culinary suffering is caused by one thing: Anna optimising the
uncommon case.

## The Common Case Is The Most Important

It can be difficult for a software engineer to focus on optimising
things that will yield the largest reward. I often struggle myself.
Instinctively, we're drawn to the most complex and interesting areas
because they're the most fun, and the most important code isn't always
the most interesting.

Now, Anna is really good at organising and she enjoys it a lot. But
even though her kitchen is always neat and tidy, it doesn't mean it's
organisesd in a way that will have the most benefit for the person
doing the cooking.

Who cares if the good china is sorted by size and
colour if your one and only chef's knife is stored in a cupboard so
high you need ladders to reach it every time?

## Find The Common Case

If Anna wants to know which utensils are the most important she could
just ask Jim. He's probably in tune enough with his cooking habits to
answer accurately. But a more reliable method would be to observe him
cooking monitor which ones he uses the most.

It's the same thing with software. Engineers can ask their users what
workflows or features they use the most (or worse, rely on their own
intuition) but a far better way is to gather this data automatically
through [profiling]({{ site.baseurl }}{%post_url
2016-12-19-a-kernel-devs-approach-to-improving%}) or monitoring, or both.

## Watch Out For Changes

Of course, things rarely stand still and it's important to filter out
the change that's just noise from the change that has significance for
day-to-day operations. Every once in a while Jim gets a present that
replaces one of his faithful tools, and then it makes total sense to
give that new gadget pride of place in the kitchen because it's going
to be used more than 95% of the other gadgets.

The optimisations of yesterday might no longer be applicable if your
code base or user behaviour has changed significantly. You need to be
vigilante and aware of when the [common case has
changed](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=58122bf1d856a4ea9581d62a07c557d997d46a19)
and has been replaced with something new.

Correct optimisation isn't always about working on really hard and
interesting problems, in fact, more often than not, it's the really
mundane code that offer the most gain when optimised.

It's unfortunate that we can't always be designing new algorithms or
writing hand-crafted assembly, but then, real life isn't all blenders and
asparagus peelers either.
