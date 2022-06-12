---
layout: post
title:  "Find and fix bugs in Linux kernel with static analysis tools"
date:   2022-05-19 11:11:51 +0530
categories: linux, C
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>
In this post, we will learn how to use static analysis tools to find various types of bugs in the linux kernel. We'll learn about scan-build (Clang's static analyzer), gcc's static analyzer, smatch, Coccinelle, sparse to find and fix bugs in the Linux kernel.

<h4><b><u>Setting up the linux kernel source</u></b></h4>
To run static analysis on the kernel we need the source code for the linux kernel. You can download any of the trees maintained by various maintainers. I am currently working with the stable tree, so I am using that for this post.
```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
```

<h4><b><u>Clang's scan build</u></b></h4>
We need two packages to be able to use scan-build - clang and clang-tools. Note that clang's version should be 11 and above.
```bash
sudo apt install clang-11 clang-tools-11
```
The linux kernel uses gcc by default, but we need to use clang as scan-build is built to work with clang. 
```bash
// To compile the entire kernel
scan-build-11 --use-cc=/usr/bin/clang-11 make CC=clang -j4 all

// To compile individual subsystem (replace fs with the subsystem name - ensure that it has a makefile)
scan-build-11 --use-cc=/usr/bin/clang-11 make CC=clang -j4 fs/
```
As the kernel is compiled, scan-build scans the code and prints out the problems that it finds on the console. You can browse through the bugs reported and fix them. 

I used scan-build on the drivers/parport driver and found a few warnings. Two of them are as follows:
```bash
drivers/parport/parport_pc.c:510:4: warning: Value stored to 'ret' is never read [deadcode.DeadStores]
                        ret = 0;
                        ^     ~
drivers/parport/parport_pc.c:639:3: warning: Value stored to 'ret' is never read [deadcode.DeadStores]
                ret = 0;

```
There is a chance that these warnings might be false positives, so make sure to read the code and understand it well before making the changes. In this case, the warning are valid, so I just removed the 'ret = 0;' lines. Now to ensure that the bug is fixed, just run the scan-build command from above again and make sure that the warning does not show up.

<br>
<h4><b><u>GCC's static analyzer</u></b></h4>
GCC has implemented a static analyzer pass which can be enabled with the '-fanalyzer' flag. To make use of this, we have to use GCC version 10 or above. To compile the kernel with this new flag, we have to add it the Makefile as follows:

```c
KBUILD_CFLAGS   := -Wall -Wundef -Werror=strict-prototypes -Wno-trigraphs \
                    -fno-strict-aliasing -fno-common -fshort-wchar -fno-PIE \
                    -Werror=implicit-function-declaration -Werror=implicit-int \
                    -Werror=return-type -Wno-format-security \
                    -fanalyzer \
                    -std=gnu11
```
Now compile the kernel and save the stderr output in another file
```c
  make -j4 2>analysis.txt
```
In the output file, you'll find all the issues detected by gcc.

<br>
<h4><b><u>Smatch</u></b></h4>
Smatch is a static analysis tool that has been built on top of sparse. Sparse provides functionality to simplify AST of a c program so that particular features of the program stand out, but it does not provide data flow analysis. This feature is provided by smatch. To install and setup smatch, follow the below steps:
```bash
sudo apt install libxml2-dev sqlite3 libgtk-3-dev llvm
git clone git://repo.or.cz/smatch.git --depth 1
cd smatch
make
```

Now, to run smatch on linux kernel, follow the following steps:
```bash
cd <path_to_kernel>
<path_to_smatch>/smatch_scripts/test_kernel.sh
```
All the warnings will be saved automatically to the file smatch_warns.txt
Also, you can run smatch on an individual file to make sure that the patches sent do not have silly mistakes.
```bash
<path_to_smatch>/smatch_scripts/kchecker --spammy <modified_c_file>
```

<br>
<h4><b><u>Coccinelle</u></b></h4>
Coccinelle is a static analysis tools and it is also used for pattern matching and text transformation such as renaming functions, adding arguments to functions, etc. To install it, run the below commands:
```bash
sudo add-apt-repository ppa:npalix/coccinelle   (Note that on Ubuntu 20.04, coccinelle is not available in default repos)
sudo apt update
sudo apt install coccinelle
```
Now to find and fix the issues which coccinelle can find, we can do it as follows:
```bash
make coccicheck MODE=report
```

<br>
<h4><b><u>Sparse</u></b></h4>
Sparse is a semantic checker that can find troublesome patterns in the code. It is also useful for navigating large codebases. We can install it as follows:
```bash
git clone git://git.kernel.org/pub/scm/devel/sparse/sparse.git
cd sparse
make && make install 
```

To run sparse on the kernel code base, we can do the following:
```bash
# Run Sparse on files being compiled
make C=1 

# Run Sparse on all files
make C=2 

# Run Sparse on driver/video directory
make C=2 drivers/video/ 
```

As mentioned before, read the code to make sure the warnings reported are not false positives. Happy hacking!

<h4><u>References and further reading</u></h4>
1. [clang scan docs](https://clang-analyzer.llvm.org/scan-build.html)
2. [GCC's fanalyzer flag](https://developers.redhat.com/blog/2020/03/26/static-analysis-in-gcc-10)
3. [Smatch LWN article] (https://lwn.net/Articles/691882/)
4. [Coccinell docs](https://www.kernel.org/doc/html/v4.15/dev-tools/coccinelle.html)
5. [Sparse's LWN article](https://lwn.net/Articles/689907/)