---
layout: post
title:  "Understanding Livepatching in linux kernel"
date:   2022-10-08 00:01:51 +0530
categories: linux, C
---

In this blog post, we will explore and learn about the livepatching feature provided by the linux kernel. We will learn about it with the help of an example livepatch that can be used in the real world. 

#### <u>What is Livepatching and why is it needed?</u>
In the server market, Linux is the dominant OS. The most common distibutions used on the server side are RHEL, Cent OS, Suse enterprise linux, Ubuntu server, etc. When it comes to maintainance tasks like upgrading kernel and upgrading software packages, organizations usually do it at regular intervals at predefined times, so that production workloads are not impacted. However, security flaws and other bugs are discovered in the kernel frequently and some of them are critical enough to warrant immediate attention. In such cases, if system administrators patch the kernels outside of their organization's predefined schedule, it might impact the production workloads and also introduce additional bugs if the organization uses a modified kernel.

For such situations, the livepatching feature was introduced in the linux kernel. Livepatching is a mechanism which can enable users to apply patches to the linux kernel without rebooting the machine. At its core, livepatching works by redirecting the code from the obsolete function to the new replacement function using the [ftrace (function trace)](https://www.kernel.org/doc/Documentation/trace/ftrace.txt) tool.

#### <u>Example</u>
We'll see the working of livepatching with the help of an example. Let's look at the capable() function defined in kernel/capability.c :
```c
bool capable(int cap)
{
	return ns_capable(&init_user_ns, cap);
}
```
In this function, there is no check to see if the capability passed as argument is in the valid range. I coded a module that called capable(99999999) and this results in a kernel bug:
```bash
[  504.249034] capability: capable() called with invalid cap=99999999
[  504.249048] ------------[ cut here ]------------
[  504.249049] kernel BUG at kernel/capability.c:372!
```

Now, we can add a check to the capable() function to check the range of number we received. For this, we have to first see how to define a livepatch.

1. <u>Define a module skeleton</u> <br>
A livepatch can be applied as a module, so define a basic module skeleton first.

2. <u>Define a new implementation of the function to be patched</u> <br>
In our case, the capable function must be defined like this:
```c
static bool livepatch_capable(int cap)
{
        pr_debug("Livepatched version of capable()");
        if (cap<0 || cap > 40) {
                pr_err("Invalid capability\n");
                return false;
        }
        return ns_capable(&init_user_ns, cap);
}
```

3. <u>Define the struct klp_func</u> <br>
From the [kernel docs](https://docs.kernel.org/livepatch/livepatch.html) - "struct klp_func is defined for each patched function. It describes the relation between the original and the new implementation of a particular function."
```c
static struct klp_func funcs[] = {
        {
                .old_name = "capable",
                .new_func = livepatch_capable,
        }, { }
};
```

4. <u>Define the struct klp_object</u> <br>
We can define an array of patched functions in the same klp_object.
```c
static struct klp_object objs[] = {
        {
                /* name being NULL means vmlinux */
                .funcs = funcs,
        }, { }
};
```

5. <u>Define the struct klp_patch</u> <br>
This defines an array of patched objects. This is the final patch that will be applied to the kernel.
```c
static struct klp_patch patch = {
        .mod = THIS_MODULE,
        .objs = objs,
};
```

6. <u>Compiling and running the patch</u> <br>
Now write the Makefile and insert the module.
```Makefile
obj-m += livepatch.o
CFLAGS_livepatch.o := -DDEBUG
all:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
```
On running the module with invalid capability, I got an error in dmesg instead of a BUG()
```bash
[ 1295.822745] Livepatched version of capable()
[ 1295.822747] Invalid capability
```

#### <u>Complete code</u>
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/livepatch.h>

#include <linux/seq_file.h>
static bool livepatch_capable(int cap)
{
        pr_debug("Livepatched version of capable()");
        if (cap<0 || cap > 40) {
                pr_err("Invalid capability\n");
                return false;
        }
        return ns_capable(&init_user_ns, cap);
}

static struct klp_func funcs[] = {
        {
                .old_name = "capable",
                .new_func = livepatch_capable,
        }, { }
};

static struct klp_object objs[] = {
        {
                /* name being NULL means vmlinux */
                .funcs = funcs,
        }, { }
};

static struct klp_patch patch = {
        .mod = THIS_MODULE,
        .objs = objs,
};

static int livepatch_init(void)
{
        return klp_enable_patch(&patch);
}

static void livepatch_exit(void)
{
}

module_init(livepatch_init);
module_exit(livepatch_exit);
MODULE_LICENSE("GPL");
MODULE_INFO(livepatch, "Y");
```

<h4><b><u>References and further reading</u></b></h4>
1. [Kernel docs](https://docs.kernel.org/livepatch)
2. [Red hat's blogpost on livepatching](https://www.redhat.com/en/topics/linux/what-is-linux-kernel-live-patching)
3. [lwn.net's articles on livepatches](https://lwn.net/Kernel/Index/#Live_patching)

