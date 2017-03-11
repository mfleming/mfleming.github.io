---
title: When Was The Last Time You Were In "The Zone"?
layout: post
comments: true
---

By pure coincidence, [two of](https://www.amazon.co.uk/Peopleware-Productive-Projects-Teams-3rd/dp/0321934113/ref=sr_1_1?ie=UTF8&qid=1489005247&sr=8-1&keywords=peopleware)
[the books](https://www.amazon.co.uk/Founders-Work-Stories-Startups-Problem-Solution/dp/1430210788/ref=sr_1_1?ie=UTF8&qid=1489005269&sr=8-1&keywords=founders+at+work)
I've recently read discussed the topic of *the zone* or *flow* -- that
state when a software developer shuts out the entire world to focus
squarely on the problem in front of them. Everyone has heard stories
like how, after solving the puzzle and finally lifting their gaze from
the screen, a programmer realises they've worked through the night
without ever having gotten up from their chair.

I think most developers have experienced it at some point. It's a
euphoric feeling while you're in it.

But *flow* is often talked about as if it's the ultimate goal for
software engineers, almost like working solely on a single problem
until it's complete is the one true measure of success. Any obstacles
to achieving it are viewed as the most egregious productivity killers.

If it's so important, why aren't we all constantly in *the zone*?

### Maybe it's environmental? ###

There's definitely evidence that environment plays a big part in
achieving *flow*. It's the whole reason for [private
offices](https://www.joelonsoftware.com/2006/07/30/private-offices-redux/)
at Fog Creek.

But that can't be the whole story. I work remotely from my home
office, wherein I've created, I think, a productive workspace. It's
free from distractions most of the time, and certainly comfortable
enough to allow me to concentrate for extended periods.

In thinking about the last time I felt like I was *in the zone*, I
remember I was writing code to automate booting Linux with various EFI
configurations (boot loaders, kernel parameters, etc). I wrote a
simple framework from scratch that parsed the git branch name to
figure out which tests to run. In other words, it was pure code
creation, written by a single person (me), without the prospect of
being integrated into some larger project. It was an interesting
problem (at least I thought so) and I could keep all the pieces in my
head as I went along.

Like most engineers, this isn't normally the kind of thing I spend my
days doing. There's always academic papers and blogs to read, meetings
to attend, and other people's code to review. Even when you're working
on your own patches, it's unusual to get the opportunity to creatively
solve a problem for hours on end.

### Are some coding tasks incompatible with *the zone*? ###

Take the job of working with the Linux kernel community. I don't think
I've ever felt in the zone while writing kernel patches. I'm not
entirely sure why that is the case. Perhaps it's because you
constantly need to reference other code and industry specifications.

When working with open source code, most of the time you're actually
integrating into an existing code base. Even if your patch series is
thousands of lines of new code, you're still using APIs that I'm
guessing you had to look up because you couldn't remember how many
arguments one of the functions took. Then you realised that, actually,
that function would benefit from being refactored. So you go and do
that and all while trying to think how every patch fits into the
larger picture of the project as a whole.

Maybe it's because many modern open source projects are so complex
that it's not possible to achieve that point where things just click
and code flows from your fingers because you're constantly
interrupting yourself to go look at something other than the code you
just wrote.

### Welcome to the No-zone Zone ###

I think it's OK that as developers we're not constantly operating in a
heightened state of euphoric code creation. Not all tasks are suited
to that kind of deeply myopic view.

In fact, some of the everyday tasks for open source project members
would be far more difficult if those developers didn't look up from
their screens every once in a while and make holistic decisions about
projects, technologies and communities.

Maybe we should stop treating *flow* like the end game and instead
just enjoy it when we find it.
