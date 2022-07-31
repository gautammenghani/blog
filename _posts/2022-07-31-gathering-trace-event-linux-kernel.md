---
layout: post
title:  "Gather trace events in linux kernel"
date:   2022-07-31 11:11:51 +0530
categories: linux, C
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>
In this short post, we will learn how to use the [ftrace tracing framework](https://www.kernel.org/doc/Documentation/trace/ftrace.txt) to gather the events generated in the linux kernel. These events can be very helpful when trying to troubleshoot a kernel bug.

<h4><b><u>Enabling ftrace</u></b></h4>
In order to be able to use ftrace, we need to enable it in the kernel's .config file. We can do it by adding the following lines to the config file

```bash
CONFIG_FUNCTION_TRACER=y
CONFIG_FUNCTION_GRAPH_TRACER=y
CONFIG_STACK_TRACER=y
CONFIG_DYNAMIC_FTRACE=y
```

<h4><b><u>Configure the tracing options</u></b></h4>
Ftrace supports gathering traces via multiple tracers - the available tracers can be listed as follows:

```bash
$ cd /sys/kernel/debug/tracing/
$ cat available_tracers
blk function_graph function nop
```
Select the tracer which can be useful for the issue at hand, and set it as current tracer.

```bash
echo function_graph > current_tracer
````
Next, we have to set the events that have to be captured. This is easy enough as we can use pattern matching to get all the needed events. For example, to trace the functions defined for the block layer, we can trace them as follows:

```bash
echo "*blk*" > set_ftrace_filter
```
The above will get all the function names containing the word "blk" and put it in the file set_ftrace_filter.

<h4><b><u>Capturing the trace events</u></b></h4>
Now the events will be captured in the tracing buffer. We need to read from there as follows:

```bash
# 1. Read the trace file
cat trace

# 2. Stream from trace_pipe file
cat trace_pipe | tee traced_events.txts
```
With the second option above, we can get the events traced in a text file as output. Now to read the ftrace events, we can just read the text file or we can use a graphical tool such as [kernelshark](https://kernelshark.org/). 

Sample output from my machine
```bash
root@syzkaller:/sys/kernel/debug/tracing# cat trace_pipe | less
 1)   9.588 us    |          blk_mq_put_tag();
 1)   5.831 us    |          blk_mq_put_tag();
 1)               |          __blk_mq_sched_restart() {
 1)               |            blk_mq_run_hw_queue() {
 1)   6.442 us    |              blk_mq_hctx_has_pending();
 1) + 18.635 us   |            }
 1) + 29.836 us   |          }
 1)   6.923 us    |          blk_queue_exit();
 1) + 86.432 us   |        } /* __blk_mq_free_request */
 1) ! 103.424 us  |      } /* blk_mq_free_request */
 ------------------------------------------
 1)  kworker-6382  =>  jbd2/sd-2889 
 ------------------------------------------

 1) + 18.315 us   |  blk_start_plug();
 1) + 63.269 us   |  __getblk_gfp();
 1)               |  bio_associate_blkg() {
 1) + 11.832 us   |    kthread_blkcg();
 1) + 12.433 us   |    blkcg_css.part.0();
 1) + 32.811 us   |    bio_associate_blkg_from_css();
 1) ! 112.882 us  |  }
 1) + 13.295 us   |  blk_cgroup_bio_start();
 1)               |  blk_mq_submit_bio() {

```
This gathered trace can now be used to understand the flow of the code and debug bugs.

<h4><u>References and further reading</u></h4>
1. [Embeddedbits article on ftrace](https://embeddedbits.org/tracing-the-linux-kernel-with-ftrace/)
2. [Kernel recipe talk by Steven Rostedt](https://www.youtube.com/watch?v=2ff-7UTg5rE)
3. [Ftrace docs](https://www.kernel.org/doc/Documentation/trace/ftrace.txt)
4. [Julia Evan's blog on trace](https://jvns.ca/blog/2017/03/19/getting-started-with-ftrace/)