---
title: "FOSForums 2026: talking about page_owner"
date: 2026-09-02
tags:
- igalia
- page_owner
---

Last week I attended FOSForums 2026 and presented about Linux `page_owner`.

# The conference

[FOSForums](https://fosforums.org) (Free and Open Source Forums) is a free conference focused on free and open source software, hardware, and the communities around them. 

Now in its 4th edition, since 2023 it has been held at the Institute of Computing at Unicamp -- one of Brazil's top public research universities -- and has featured speakers from Igalia, Red Hat, Linaro, RISC-V International, Toradex, Unicamp (State University of Campinas), USP (University of São Paulo), Eldorado Research Institute, and others.

The partners of the conference for this year are [RISC-V International](https://riscv.org), [Unicamp](https://www.unicamp.br/en/) and its [Institute of Computing](https://ic.unicamp.br/en/) and [Institute for the Conservation of Free Technologies](https://www.ictl.org.br/en/).

It's still a relatively new and small conference -- around 30 attendees this year -- but a notably high share of them are students, both undergraduate and graduate. That makes it an especially relevant place to share knowledge, identify talent, and support people early in their tech careers -- contributing back to the community.

This year's schedule covered the FinOps Foundation, open science, continuous integration with RISC-V/Linux/KernelCI, Linux filesystem performance on flash devices, and Linux `page_owner` (my talk), plus a hands-on session with BitDogLab (an open-source educational hardware platform). A talk on turning bug reports into kernel patches was also planned but unfortunately couldn't be delivered -- keep an eye on Heitor's [blog](https://blog.ctrlbyte.com)!

![FOSForums 2026: BitDogLab's board and conference logo](/images/fosforums-2026/fosforums-2026-picture-board-logo.jpg)

# The talk

Picking up the thread on `page_owner` (see the blog post [series]({{< ref "tags/page_owner" >}})), I wanted to put together a more introductory, user-friendly version of the talk _[Improving `page_owner` for profiling and monitoring memory usage per allocation stack trace](http://www.youtube.com/watch?v=qFdjO3t5F9I)_ presented at [Linux Plumbers Conference 2025](https://lpc.events/event/19/contributions/2202/).

I opened with an introduction about Igalia -- _who we are_, _what we work with_, and _what we do_ -- and how to _join us_ through our [open positions](https://www.igalia.com/jobs) and our [Coding Experience](https://www.igalia.com/coding-experience) program.

The technical part covered the problem at hand, "_Where is memory going?_", and then walked through `page_owner` from definition to application: what it is, what it is _for_, how it works, how to use it, then basic and advanced examples.

The session recording is available on YouTube (currently within the [live stream](https://www.youtube.com/live/Uq6HAXrwA_Q?t=28077s); later in the FOSForums 2026 [playlist](https://www.youtube.com/playlist?list=PLe8NBbBrwvJ4)), and the slides are available [here](/slides/fosforums-2026-page_owner.pdf).

![FOSForums 2026: mfo's talk; photo by mhcerri@igalia.com](/images/fosforums-2026/fosforums-2026-picture-mfo-talk.jpg)

# Acknowledgments

Thank you to the FOSForums organizing committee for the great conference, and Igalia for the opportunity to present this work.

![FOSForums 2026: Igalia backpack and audience](/images/fosforums-2026/fosforums-2026-picture-igalia-backpack.jpg)
