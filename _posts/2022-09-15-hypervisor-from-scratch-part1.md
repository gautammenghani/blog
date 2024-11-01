---
layout: post
title:  "Building hypervisor from scratch part 1"
date:   2022-09-15 00:01:51 +0530
categories: linux, C, hypervisor
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>

In this blog series, we'll be creating a type 1 (bare metal) hypervisor from scratch. We'll be creating a Linux kernel module that will function as a VMM (Virtual Machine monitor) for the AMD CPU's native SVM functionality. This blog series is inspired by the [excellent series by rayanfam.com](https://rayanfam.com/topics/hypervisor-from-scratch-part-1) where they built a KVM like module for windows on Intel CPUs. We'll build a KVM like module for AMD CPUs on linux. In this post, we'll first get an overview of how KVM works and then we'll understand how to detect and enable AMD's virtualization operation. Let's go!

<h4><b><u>Working of KVM</u></b></h4>

KVM stands for Kernel Virtual Machine. It is a module in linux that has a common API but architecture specific implementation, since every CPU implements virtualization features differently. At a high level, users use emulators like QEMU to use virtual machines and QEMU then uses KVM to accelerate the performance of the virtual machines. KVM exposes a character file "/dev/kvm" against which, hypervisors/emulators like QEMU can fire IOCTLs like KVM\_CREATE\_VM, KVM\_CREATE\_VM, KVM\_RUN, etc. The KVM module receives these IOCTL requests and in turn uses the virtualization specific assembly instructions to run and manage the virtual machines.
With a basic understanding of KVM architecture in place, let's start with writing our own KVM like module.

<h4><b><u>Checking if CPU supports virtualization</u></b></h4>
Let's start by checking the CPU model - we need to make sure we know the underlying CPU as there are differences between the virtualization technologies used by different processors.
```c
// Get the vendor name in string format
cpu = cpu_data(0);
vendor_name = cpu.x86_vendor_id;

// Get the model number of the processor
pr_warn("%d", boot_cpu_data.x86_model);
```

Now that we have the vendor name, we can continue if the CPU is AMD and abort otherwise. Let's also check if SVM is supported in the CPU.
```c
pr_warn("SVM supported? :  %s\n", boot_cpu_has(X86_FEATURE_SVM) == 1?"Yes":"No");
```
In the above line, boot_cpu_has() is a macro which in our case checks if the CPU has the SVM [flag](https://www.kernel.org/doc/html/latest/x86/cpuinfo.html) set.

We know now if SVM is supported, so the next step is to check if the SVME bit is set. For this, we can use MSRs (Model specific Registers). In our case, we can use AMD's Extended Feature enable register (EFER) in which the 12th bit (starting from 0) indicates if SVM is enabled. The Linux kernel does provide APIs for this, although we can also get this information from running assembly instructions. Let's read this register and see if the SVME bit is enabled.
```c
static void enable_svme_bit (void *data)
{
        int efer;

        rdmsrl(MSR_EFER, efer);
        wrmsrl(MSR_EFER, (efer | EFER_SVME));
}


static int __init gmh_init(void)
{
.
.
.
        if (!strncmp (vendor_name, "AuthenticAMD", strlen(vendor_name))) {
                pr_warn("SVM supported? :  %s\n", boot_cpu_has(X86_FEATURE_SVM) == 1?"Yes":"No");
                rdmsrl(MSR_EFER, efer);
                if (efer & EFER_SVME){
                        pr_warn("EFER_SVME is already set\n");
                        return -EBUSY;
                }   
        }   
        else if (!strncmp (vendor_name, "GenuineIntel", strlen(vendor_name)))
                pr_warn("VMX supported? :  %s\n", boot_cpu_has(X86_FEATURE_VMX) == 1?"Yes":"No");
        else
                pr_warn("Virtualization is not supported\n");
        if (register_chrdev(gmh_major, "gmh", &gmh_fops) < 0) {
                pr_debug("cannot obtain major number: %d\n", gmh_major);
        }   
        cls = class_create(THIS_MODULE, "gmh");
        device_create(cls, NULL, MKDEV(gmh_major, 0), NULL, "gmh");

        on_each_cpu(enable_svme_bit, NULL, 1);
}
```
In the above code, we first check if the SVME bit is enabled and if it is not enabled, we set it for all cpus so that we put the CPU in SVM operation and can use other SVM operations. We also create a char device "/dev/gmh" for userspace to be able to fire IOCTLs to our module. The code for this part can be found [here](https://github.com/gautammenghani/hypervisor-from-scratch/tree/main/part1).

In the next post, we'll learn how to setup our own VM. Stay tuned.

<h4><u>References and further reading</u></h4>
1. [AMD's systems programming manual](https://www.amd.com/system/files/TechDocs/24593.pdf)
2. [Intel's reference manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
3. [Intel/AMD CPU internals](https://github.com/LordNoteworthy/cpu-internals)

