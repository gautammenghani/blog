---
layout: post
title:  "Writing a device driver in linux"
date:   2022-04-30 11:11:51 +0530
categories: linux, C
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>
In this post, we will learn how to write a character device driver in linux. We will create a simple device driver which prints the the number of times each of the functions is called.

<h4><u>General architecture of a device driver</u></h4>
<img src="{{ site.baseurl }}/assets/images/6_1_general_arch.jpg" alt="General architecture">
Note that we are not using a physical hardware device, just for simplicity's sake.

<h4><u>Initializing the module</u></h4>
Every module written for the kernel needs to be initialized with the kernel first. This is typically done by writing a custom function and registering with the kernel with module_init() function
```c
static int __init chardevice_init(void)
{
        major = register_chrdev(0, "chardevice", &fops);
        if (major < 0) {
                pr_alert("Registering device failed : %d\n", major);
                return major;
        }

        printk(KERN_ALERT "Major number assigned is : %d\n", major);
        cls = class_create(THIS_MODULE, "chardevice");
        device_create(cls, NULL, MKDEV(major, 0), NULL, "chardevice");
        return 0;
}
module_init(chardevice_init);
```
1. In the above code we first need a major number for our driver, which we can get by using register_chrdev() function. The first argument is the major number we need, in this case 0 means the kernel allocates it dynamically. The second argument is the device name and the third argument is a pointer to file operations structure which we will learn about shortly.
2. At this point our driver is registered and we need to expose it over a file in the /dev directory. We do this with the device_create() function. 

<h4><u>Creating the file operations structure</u></h4>
The file operations structure is a kernel defined structure that holds pointers to the functions that are implemented by the driver to perform various operations on the device such as open, read, write, etc. Let us look at our functions.
```c
static struct file_operations "fops" = {
        .read = device_read,
        .write = device_write,
        .open = device_open,
        .release = device_release
};

static int device_open(struct inode *inode, struct file *file)
{
    static int counter = 0;
    printk(KERN_ALERT "open called :  %d times\n", counter++);
    try_module_get(THIS_MODULE); // Increment the count of processes using this module
    return 0;
}

static int device_release(struct inode *inode, struct file *file)
{
    module_put(THIS_MODULE); // Decrement the count of processes using this module
    return 0;
}

static ssize_t device_read(struct file *filp,  char __user *buffer, 
                           size_t length, loff_t *offset)
{
    static int counter = 0;
    printk(KERN_ALERT "read called :  %d times\n", counter++);
    return 0;
}

static ssize_t device_write(struct file *filp, const char __user *buff,
                            size_t len, loff_t *off)
{
        static int counter = 0;
        printk(KERN_ALERT "write called :  %d times\n", counter++);
        return 1;

}
```

<h4><u>Cleaning up</u></h4>
A module is to be exited after use. We define a cleanup function where we can perform cleanup tasks like releasing resources. The code for the same is as follows
```c
static void __exit chardevice_exit(void)
{
        device_destroy(cls, MKDEV(major, 0));
        class_destroy(cls);
        unregister_chrdev(major, "chardevice");
}
module_exit(chardevice_exit);
```

<h4><u>Compiling the module</u></h4>
We have to write a makefile to compile the module. We will get a .ko file which we load into the kernel with insmod command.
```Makefile
ifneq ($(KERNELRELEASE),)
        obj-m := chardevice.o
# Otherwise we were called directly from the command
# line; invoke the kernel build system.
else
        KERNELDIR ?= /lib/modules/`uname -r`/build
        PWD := $(shell pwd)
default:
        $(MAKE) -C $(KERNELDIR) M=$(PWD) modules
endif
```

<h4><u>Talking to the device</u></h4>
Now we will write a c program to talk to the file and perform the operations. 
```c
#include<stdio.h>
int main(){
        FILE *fp;
        char data[80] = "hacking kernel";
        fp = fopen("/dev/chardevice", "w+");
        if (fp==NULL)
                printf("go learn basic shit");
        else {
                while (fgets(data, 50, fp)!=NULL) {
                        printf("%s", data);
                }
        }
        char c = 'c';
        fwrite(&c, 1, sizeof c, fp);
        fclose(fp);
        return 0;
}
```
Now let's see the output in dmesg
```dmesg
[21851.342331] open called :  0 times
[21851.342341] write called :  0 times
[21880.844521] open called :  1 times
[21880.844531] write called :  1 times
```

Here's the entire code
```c
#include <linux/cdev.h>
#include <linux/delay.h>
#include <linux/device.h>
#include <linux/fs.h>
#include <linux/init.h>
#include <linux/irq.h>
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/poll.h>

static int device_open(struct inode *, struct file *);
static int device_release(struct inode *, struct file *);
static ssize_t device_read(struct file *, char __user *, size_t, loff_t *);
static ssize_t device_write(struct file *, const char __user *, size_t,loff_t *);

static int major;
static struct class *cls;
MODULE_LICENSE("Dual BSD/GPL");

static struct file_operations "fops" = {
        .read = device_read,
        .write = device_write,
        .open = device_open,
        .release = device_release
};

static int device_open(struct inode *inode, struct file *file)
{
    static int counter = 0;
    printk(KERN_ALERT "open called :  %d times\n", counter++);
    try_module_get(THIS_MODULE);
    return 0;
}

static int device_release(struct inode *inode, struct file *file)
{
    module_put(THIS_MODULE);
    return 0;
}

static ssize_t device_read(struct file *filp, /* see include/linux/fs.h   */
                           char __user *buffer, /* buffer to fill with data */
                           size_t length, /* length of the buffer     */
                           loff_t *offset)
{
    static int counter = 0;
    printk(KERN_ALERT "read called :  %d times\n", counter++);
    return 0;
}

static ssize_t device_write(struct file *filp, const char __user *buff,
                            size_t len, loff_t *off)
{
        static int counter = 0;
        printk(KERN_ALERT "write called :  %d times\n", counter++);
        return 5;

}

static int __init chardev_init(void)
{
        major = register_chrdev(0, "chardevice", &"fops");
        if (major < 0) {
                pr_alert("Registering char device failed : %d\n", major);
                return major;
        }

        pr_info("Major number assigned : %d\n", major);
        cls = class_create(THIS_MODULE, "chardevice");
        device_create(cls, NULL, MKDEV(major, 0), NULL, "chardevice");
        pr_info("Device created on /dev/%s\n", "chardevice");
        return 0;
}
static void __exit chardev_exit(void)
{
        device_destroy(cls, MKDEV(major, 0));
        class_destroy(cls);
        unregister_chrdev(major, "chardevice");
}
module_init(chardev_init);
module_exit(chardev_exit);
```

<h4><u>References and further reading</u></h4>
1. [Operating systems - Three easy pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/)
2. [Linux kernel module programming guide](https://sysprog21.github.io/lkmpg)
