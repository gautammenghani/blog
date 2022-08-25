---
layout: post
title:  "Linux kernel mentorship experience - summer 2022"
date:   2022-08-24 00:01:51 +0530
categories: linux, C
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>

Around the start of May 2022, I was browsing through the mentorships on the linux foundation site, and I came across the Linux kernel bug fixing mentorship summer 2022. I had just decided to learn more about the linux kernel, so this was a great chance to proceed ahead on my goal. I applied to the program and was selected after the initial screening round. I will share my experience of being a part of the 3 month program. 

In the initial part of the program, I started by learning about event tracing, static analysis, dynamic analysis and fuzzing, since these techniques are the main techniques used for finding and fixing bugs. Once I was familiar with the basic debugging techniques, I started looking for the bugs. Since the kernel is a huge and complex codebase, our mentors - [Shuah Khan](https://www.linkedin.com/in/shuah-khan) and [Pavel Skripkin](https://pskrgag.github.io/about/) suggested us to explore all the subsystems and then start working on our areas of interest. So, I started exploring various areas of the kernel and contributed by fixing static analysis warnings in various parts of the kernel. Along with this, I was also exploring the various subsystems of the kernel. Finally, I decided to work on kselftests, memory management, and syzbot bugs. 

I started to work on the kselftests by running each selftest and trying to find bugs and missed cases. I started by improving documentation and error messages. Gradually, as I learned more, I started looking the tests for each subsystem. When I attempted to run the damon tests, I noticed that all the tests were skipped without providing the reason. So I started digging and figured out the cause. I sent a patch to add the new tests and it was [accepted](https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/commit/?id=43fe0cc46b6206b25f0f13bb249f0078441ae15a).

Next, I started looking at syzbot bugs. I scanned through many bugs before I came across a bug which I thought I could work on. The bug involved the block layer and the nbd driver. So I started learning about the block layer - how it handles the bio requests, the working of blk_mq layer and so on. I worked on the bug for a few weeks, but all the approaches I tried led me into rabbit holes. Since it looked to be a complex race condition, I decided to learn more before giving it a shot again. 

So I continued working with kselftests and damon - data access monitoring framework to improve memory management performance. For kselftests, I decided to look at the TODOs that have been left there by other developers. I worked on refactoring a piece of code in ipsec tests in selftests/net, which was responsible for checking the cryptography algorithm used. I sent out a [patch](https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/commit/?id=93d7c52a6eb93e58e4569bd4de95ba3b19e3cf20) for this, which was accepted. Next, I wanted to contribute to damon, so I started looking at damon tests. I noticed that some of the tests are located out of the tree, so I started learning about them. I worked on [adding hugetlb support to masim](https://github.com/sjp38/masim/pull/3), which is important to add hugepage tests to [damon-tests](https://github.com/awslabs/damon-tests).

Overall, the Linux kernel mentorship program has been an excellent learning opportunity for me, especially with awesome mentors who guided us throughout the program. Furthermore, the kernel community is very welcoming to new developers, which definitely is an encouraging aspect. I learned a lot, identified my areas of interest and I aim to continue to improve on them by working as a kernel developer.
