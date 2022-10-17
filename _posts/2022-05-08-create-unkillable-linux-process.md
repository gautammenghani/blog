---
layout: post
title:  "Creating unkillable processes in linux"
date:   2022-05-08 11:11:51 +0530
categories: linux, C
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>
In this post, we will learn how to make an userspace process unkillable. We will create a simple character device driver which accepts the pid (process id) of a process and makes it unkillable. The source code for this can be found on [my github](https://github.com/gautammenghani/upas).

<h4><u>Why make an unkillable process?</u></h4>
Sometimes, we may be writing a critical program that has to complete its tasks successfully and a failure to do so may be catastrophic for the system. For example, the init process in linux cannot be in a situation where a user accidently kills it.

<h4><u>What makes a process unkillable?</u></h4>
We know that the init process is unkillable. Also, a process is internally represented as a [task struct](https://stackoverflow.com/questions/56531880/how-does-the-kernel-use-task-struct) in the linux kernel. So we can start by exploring the fields available inside task struct.
While browsing the [source code for task struct](https://elixir.bootlin.com/linux/latest/source/include/linux/sched.h), I noticed the following fields:
```
/* Signal handlers: */
	struct signal_struct		*signal;
	struct sighand_struct __rcu	*sighand;
```
Since a process is generally killed with SIGINT and SIGKILL signals, the signal handlers field look promising. Let's explore this more. 

<h4><u>Exploring signal struct</u></h4>
While going through the source code of signal_struct, I noticed this line:
```
unsigned int		flags; /* see SIGNAL_* flags below */
```
Sounds useful, let's see the flags
```
/*
 * Bits in flags field of signal_struct.
 */
#define SIGNAL_STOP_STOPPED	0x00000001 /* job control stop in effect */
#define SIGNAL_STOP_CONTINUED	0x00000002 /* SIGCONT since WCONTINUED reap */
#define SIGNAL_GROUP_EXIT	0x00000004 /* group exit in progress */
/*
 * Pending notifications to parent.
 */
#define SIGNAL_CLD_STOPPED	0x00000010
#define SIGNAL_CLD_CONTINUED	0x00000020
#define SIGNAL_CLD_MASK		(SIGNAL_CLD_STOPPED|SIGNAL_CLD_CONTINUED)

#define SIGNAL_UNKILLABLE	0x00000040 /* for init: ignore fatal signals */

#define SIGNAL_STOP_MASK (SIGNAL_CLD_MASK | SIGNAL_STOP_STOPPED | \
			  SIGNAL_STOP_CONTINUED)
```
There we have it, the SIGNAL_UNKILLABLE flag should be what we want. If this flag is set for a process, it should become unkillable. 

<h4><u>Writing a character device driver</u></h4>
In my [previous post](https://gum3ng.xyz/linux,/c/2022/04/30/writing-kernel-module.html), I explained how can we write a simple character device driver. Such a device driver is useful as any userspace process will be able to call it. The following is the code for the same:

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/types.h>
#include <linux/proc_fs.h>
#include <linux/sched.h>
#include <linux/sched/signal.h>
#include <linux/pid.h>
MODULE_LICENSE("Dual BSD/GPL");

int unkill_major = 113;

ssize_t unkill_write(struct file *filp, const char *buf, size_t count, loff_t *f_pos);
ssize_t unkill_read(struct file *filp, char *buf, size_t count, loff_t *f_pos);
int unkill_open(struct inode *inode, struct file *filp);
int unkill_release(struct inode *inode, struct file *filp);


int unkill_open(struct inode *inode, struct file *filp)
{
        return 0;
}

int unkill_release(struct inode *inode, struct file *filp)
{
        return 0;
}

ssize_t unkill_read(struct file *filp, char *buf, size_t count, loff_t *f_pos)
{
        struct pid *pid_struct;
        struct task_struct *ts;

        // count is read as target pid
        printk("Unkill module got the pid : %d", (int) count);

        // get pid struct
        pid_struct = find_get_pid((int) count);

        // get the task_struct
        ts = pid_task(pid_struct, PIDTYPE_PID);


        ts->signal->flags = ts->signal->flags | SIGNAL_UNKILLABLE;
        printk("Unkillable: pid %d marked as unkill\n", (int) count);
        if (*f_pos) {
                return 0;
        } else {                
                *f_pos++;
                return 1;
        }
}

ssize_t unkill_write(struct file *filp, const char *buf, size_t count, loff_t *f_pos)
{
        return 0;
}
struct file_operations unkill_fops = {
        .read = unkill_read,
        .write = unkill_write,
        .open = unkill_open,
        .release = unkill_release
};

int unkill_init(void)
{
        if (register_chrdev(unkill_major, "unkill", &unkill_fops) < 0 ) {
                printk("cannot obtain major number %d\n", unkill_major);
                return 1;
        }

        printk("Insert unkill module\n");
        return 0;
}
void unkill_exit(void)
{
        unregister_chrdev(unkill_major, "unkill");
        printk("Removing unkill module\n");
}


module_init(unkill_init);
module_exit(unkill_exit);
```
In the above code the most important part is the read function. In the read function, we first extract the pid struct of a process from its pid. We then extract task struct from this pid struct, and finally set the SIGNAL_UNKILLABLE flag.
Now make the device file as follows:
```
sudo mknod /dev/unkill c 113 0
sudo chmod 666 /dev/unkill
```

<h4><u>Make userspace process unkillable</u></h4>
All we have to do is open the char device file and read from it.
```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main()
{
        int file_desc;
        char c;
        char buffer[10];
        int my_pid = getpid();

        printf("My pid: %d\n", my_pid);
        file_desc = open("/dev/unkill", O_RDWR);
        if (file_desc < 0)
                printf("Error opening file\n");
        printf("File opened\n");
        read(file_desc, &c, my_pid);
        printf("Process is unkillable!\n");
        read(STDIN_FILENO, buffer, 10);
        printf("exiting \n");
        return 0;
}
```
Now try killing this process with kill command. The process will not die with the kill command.

<h4><u>References and further reading</u></h4>
1. [Elixir bootlin](https://elixir.bootlin.com)
2. [Linux kernel module programming guide](https://sysprog21.github.io/lkmpg)
