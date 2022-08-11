---
layout: post
title:  "Fixing a kernel bug reported by syzkaller"
date:   2022-08-11 11:11:51 +0530
categories: linux, C
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>
In this post, we will learn how to debug and fix a bug reported by syzkaller - a kernel fuzzer that finds bugs in the kernel using dynamic analysis. We will be looking at a sample bug - [warning that was triggered in the USB core layer](https://syzkaller.appspot.com/bug?id=e378e6a51fbe6c5cc43e34f131cc9a315ef0337e) as a sample for learning.

<h4><b><u>Ensuring that the bug is reproducible</u></b></h4>
Before attempting to fix a bug, it is a good idea to see if it can be reproduced. A bug that is reproducible is relatively easier to fix, as we can check our fix to make sure we are on the right track.

To reproduce a bug, compile the kernel with the config file that is available on the syzkaller dashboard and boot it in a VM. After this, run the given reproducer to check if the bug is triggerable. In our case, the bug is triggered on each run of the reproducer, so we can move ahead.

<h4><b><u>Reading the stacktrace</u></b></h4>
When the kernel crashes, we get a stacktrace which shows the function backtrace as follows (a small part is shown):
```bash
? rcu_read_lock_sched_held+0x3a/0x70
[   94.124819][ T6497]  ? trace_kmalloc+0x35/0xf0
[   94.125249][ T6497]  send_packet+0x44b/0xbf0
[   94.125661][ T6497]  vfd_write+0x2d7/0x550
[   94.126060][ T6497]  ? send_packet+0xbf0/0xbf0
[   94.126452][ T6497]  vfs_write+0x25e/0xae0
[   94.126834][ T6497]  ksys_write+0x127/0x250
[   94.127237][ T6497]  ? __ia32_sys_read+0xa0/0xa0
[   94.127673][ T6497]  ? syscall_enter_from_user_mode+0x21/0x70
[   94.128136][ T6497]  ? syscall_enter_from_user_mode+0x21/0x70
[   94.128653][ T6497]  do_syscall_64+0x35/0xb0
[   94.129053][ T6497]  entry_SYSCALL_64_after_hwframe+0x63/0xcd
```

This is not very readable, but fortunately kernel provides a script to decode the stacktrace. We can use it as follows:
```bash
$ scripts/decode_stacktrace.sh vmlinux /home/biggie/linux < ../crashes/crash.txt > ../crashes/decode.txt
```
This gives us the line numbers along with the function names.

<br>
<h4><b><u>Finding the root cause</u></b></h4>
Let us start with the line at which the bug occurs.
```c
  if (urb->hcpriv) {
      WARN_ONCE(1, "URB %pK submitted while active\n", urb);
      return -EBUSY;
  }
```
The hcpriv field of the USB request blog is not NULL, and this causes the warning that the URB was active while submitting. Good start, let's read the functions in the backtrace.

Let's look at the code that called this function - send_packet(ictx)
```c
  reinit_completion(&ictx->tx.finished);
	ictx->tx.busy = true;
	smp_rmb(); /* ensure later readers know we're busy */

	retval = usb_submit_urb(ictx->tx_urb, GFP_KERNEL);
```
So before calling usb_submit_urb(), we set the imon_context->tx.busy to true to indicate that a write is in progress. Now let's look at the fields of the strict imon_context
```c
struct imon_context {
	struct device *dev;
	/* Newer devices have two interfaces */
	struct usb_device *usbdev_intf0;
	struct usb_device *usbdev_intf1;
        .
        .
        .        
	struct urb *tx_urb;
	bool tx_control;
	unsigned char usb_rx_buf[8];
	unsigned char usb_tx_buf[8];
	unsigned int send_packet_delay;

	struct tx_t {
		unsigned char data_buf[35];	/* user data buffer */
		struct completion finished;	/* wait for write to finish */
		bool busy;			/* write in progress */
		int status;			/* status of tx completion */
	} tx;
        .
        .
        .
        .      
};
```

So tx.busy indicates if a write is in progress, and the warning says that URB was submitted while active; we can try checking if ictx->tx.busy is true before send_packet() was called. We can do this by adding print statements in the first part of send_packet().

```c
pr_debug("ictx->tx.busy : %c\n", (ictx->tx.busy)==true?'y':'n');
pr_debug("ictx->tx.status : %d\n", (ictx->tx.status));
```

And here it is
```bash
[   64.714175][  T761] imon:send_packet: ictx->tx.busy : n
[   64.714738][  T761] imon:send_packet: ictx->tx.status : 0
[   64.978212][  T761] rc_core: IR keymap rc-imon-pad not found
[   64.978877][  T761] Registered IR keymap rc-empty
[   64.979484][  T761] imon 1-1:0.0: Looks like you're trying to use an IR protocol this device does not support
                                          .
                                          .
                                          .
                                          .                                          
[   65.170186][  T761] imon 1-1:0.0: iMON device (15c2:0040, intf0) on usb<1:2> initialized
[   65.329111][ T6375] imon:send_packet: ictx->tx.busy : n
[   65.329669][ T6375] imon:send_packet: ictx->tx.status : 0
[   65.380310][ T6395] imon:send_packet: ictx->tx.busy : y
[   65.380821][ T6395] imon:send_packet: ictx->tx.status : 0
[   65.381304][ T6395] ------------[ cut here ]------------
[   65.381785][ T6395] URB ffff8881064ce900 submitted while active
[   65.382521][ T6395] WARNING: CPU: 0 PID: 6395 at drivers/usb/core/urb.c:378 usb_submit_urb+0x14bd/0x1870
```
So the warning happens when send_packet() is called when a write is already in progress. The fix for this is easy enough:

```c
static int send_packet(struct imon_context *ictx)
{
	unsigned int pipe;
	unsigned long timeout;
	int interval = 0;
	int retval = 0;
	struct usb_ctrlrequest *control_req = NULL;

	if (ictx->tx.busy)
	    return -EBUSY;
```
With this fix in place, the reproducer no longer triggers a problem.

<br>
<h4><b><u>General tricks when working on syzkaller bugs</u></b></h4>
It is helpful to get a reproducer that can trigger the bug reliably as evident in the above example. Next, try to find out if the bug is a race condition. This can be done either with the reproducer code, or using the -smp flag in QEMU command line. Finally, try to use different tools like ftrace, kgdb, etc. depending on the nature of the bug.

<h4><u>References and further reading</u></h4>
1. [Debugging talk by Sergio Prado](https://static.sched.com/hosted_files/osseu19/09/slides.pdf)
2. [Blog posts walking through different bugs](https://gist.github.com/gautammenghani/b6d0906921c69fe7d8a17df1e62b4f89)
3. [KGDB docs](https://www.kernel.org/doc/html/v4.14/dev-tools/kgdb.html)