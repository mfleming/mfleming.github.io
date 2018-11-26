---
layout: post
title: The psychology behind why humans suck at reading performance data
---

People often think that performance testing frameworks exist because
machines are good at finding patterns in performance data and humans
are not.

Actually, humans are *very* good at finding patterns in data. In fact,
[we're too
good](https://en.wikipedia.org/wiki/Apophenia#Models_of_pattern_recognition).
We see patterns in data where none exist because our minds are
hardwired to notice them. Detecting patterns and ascribing meaning to
them was once thought to be a psychotic thought process linked with
[schizophernia](https://en.wikipedia.org/wiki/Klaus_Conrad), but
psychologists now understand it to be an evolutionary skill which
allows us to make predictions about the world we inhabit.

But those pattern recognition skills that helped our ancestors find
food in the wild have all kinds of consequences ranging from the
bizarre (sometimes we [see faces in
clouds](https://www.livescience.com/25448-pareidolia.html)) to the
downright illogical (thinking a coin that has turned up heads for the
last few flips [is likely to be heads next
time](https://en.wikipedia.org/wiki/Gambler%27s_fallacy)).

And it's because of this [cognitive
bias](https://en.wikipedia.org/wiki/Cognitive_bias) that we are really
bad at reading and comparing performance numbers without the help of
performance analysis tools -- our brains simply cannot view the data
without trying to find patterns.

For long-running performance tests, it's common to run the test case
standalone, outside of the test suite, for example when changing the
code in between runs. But if you've ever eyeballed a test result
instead of using the reporting framework of your chosen test suite,
you've potentially fallen victim to this quirk of human nature.

Measuring performance is hard. Let the machines help.
