---
layout: post
title:  "Find and fix bugs in Linux kernel with clang scan"
date:   2022-05-19 11:11:51 +0530
categories: linux, C
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>
In this post, we will learn how to use scan-build, i.e,  Clang's static analyzer to find and fix bugs in the Linux kernel.

<h4><u>Installing the necessary tools</u></h4>
We need two packages to be able to use scan-build - clang and clang-tools. Note that clang's version should be 11 and above.
```bash
sudo apt install clang-11 clang-tools-11
```

Next we need the source code for the linux kernel. You can download any of the trees maintained by various maintainers. I am currently working with the stable tree, so I am using that for this post.
```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
```

<h4><u>Compile the kernel with clang and scan build</u></h4>
The linux kernel uses gcc by default, but we need to use clang as scan-build is built to work with clang. 
```bash
// To compile the entire kernel
scan-build-11 --use-cc=/usr/bin/clang-11 make CC=clang -j4 all

// To compile individual subsystem (replace fs with the subsystem name - ensure that it has a makefile)
scan-build-11 --use-cc=/usr/bin/clang-11 make CC=clang -j4 fs/
```
As the kernel is compiled, scan-build scans the code and prints out the problems that it finds on the console. Let's take one of the bugs reported and fix it. 

<h4><u>Finding a bug and fixing it</u></h4>
I used scan-build on the drivers/parport driver and found a few warnings. Two of them are as follows:
```bash
drivers/parport/parport_pc.c:510:4: warning: Value stored to 'ret' is never read [deadcode.DeadStores]
                        ret = 0;
                        ^     ~
drivers/parport/parport_pc.c:639:3: warning: Value stored to 'ret' is never read [deadcode.DeadStores]
                ret = 0;

```

There is a chance that these warnings might be false positives, so make sure to read the code and understand it well before making the changes. In this case, the warning are valid, so I just removed the 'ret = 0;' lines. Now to ensure that the bug is fixed, just run the scan-build command from above again and make sure that the warning does not show up.


<h4><u>References and further reading</u></h4>
1. [clang scan docs](https://clang-analyzer.llvm.org/scan-build.html)
2. [https://tthtlc.wordpress.com/2018/03/21/kernel-source-code-static-analysis-with-clang-lots-of-false-positive/](https://tthtlc.wordpress.com/2018/03/21/kernel-source-code-static-analysis-with-clang-lots-of-false-positive/)
