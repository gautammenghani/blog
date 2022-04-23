---
layout: post
title:  "Using semaphores and mutex locks in linux"
date:   2022-04-21 11:11:51 +0530
categories: linux
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>
In this post, we will learn how to manage access to shared resources in linux with the concepts of semaphores and mutexes. Let us first understand why concurrent accesses can lead to unexpected results.

<h4><u>Side effects of concurrency</u></h4>
Let us consider an example that can uncover the side effects of multiple threads accessing globally shared resources. In the following program, we have a global variable called count. We have two threads, p1 and p2, whose job is to increase the value of count variable by 10 million each. 

```c
#include<stdio.h>
#include<pthread.h>
#include <unistd.h>

volatile int count=0;
void* funcA(void *arg) {
        int i;
        for(i=0;i<1e7;i++) 
                count++;
        printf("%s is done\n", (char *)arg);
        return NULL;
}

int main(int argc, char **argv) {
        pthread_t p1,p2;
        int rc;
        printf("main begins\n");
        pthread_create(&p1, NULL, funcA, "Thread p1");
        pthread_create(&p2, NULL, funcA, "Thread p2");
        pthread_join(p1, NULL);
        pthread_join(p2, NULL);
        printf("main end : %d\n",count);
        return 0;
}
```

When the above program is run, you might expect that the program will print "main end : 20000000". But let's see the real results
<img src="{{ site.baseurl }}/assets/images/5_1_inconsis_res.jpg" alt="inconsistent results">

<h4><u>Why did the we get above results?</u></h4>
To understand what really happened, we have to look at the disassembly of the program. For this, you can use either gdb or objdump
<img src="{{ site.baseurl }}/assets/images/5_2_disassembly.jpg" alt="disassembly">

Since the addition operation did not return the correct results, let us look at the lines that are responsible for addition
```asm
    0x00000000000011e2 <+25>:    mov    eax,DWORD PTR [rip+0x2e2c]        # 0x4014 <count>
    0x00000000000011e8 <+31>:    add    eax,0x1
    0x00000000000011eb <+34>:    mov    DWORD PTR [rip+0x2e23],eax        # 0x4014 <count>
```
Assume the current value of count to be 50. To add 1 to the count, the count variable is loaded into the eax register with the mov instruction. The add instruction adds 1 to it and then we move the new value back to the memory location. What exactly is the problem?
When thread A is executing the above code, it may happen that an interrupt occurs just after the add instruction due to which context switch occurs and the OS saves the state of process's state in TCB. Now when thread B starts executing, it reads the value of count as 50 as thread A did not store 51 previously. This happens continuously throughout the execution, which results in inconsistent value of count on each run.

<h4><u>How to solve this problem?</u></h4>
The above code where the value of count is incremented is called critical section. To solve the above problem, we can use the concept of mutex locking (mutual exclusion) where the critical section can only be accessed by one thread at a time. 
```c
#include<stdio.h>
#include<pthread.h>
#include <unistd.h>

volatile int count=0;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
void* funcA(void *arg) {
        int i;
        for(i=0;i<1e7;i++) {
                pthread_mutex_lock(&lock);
                count++;
                pthread_mutex_unlock(&lock);
        }
        printf("%s is done\n", (char *)arg);
        return NULL;
}
int main(int argc, char **argv) {
        pthread_t p1,p2;
        int rc;
        printf("main begins\n");
        pthread_create(&p1, NULL, funcA, "Thread p1");
        pthread_create(&p2, NULL, funcA, "Thread p2");
        pthread_join(p1, NULL);
        pthread_join(p2, NULL);
        printf("main end : %d\n",count);
        return 0;
}
```
In the above code, we initialize a mutex lock with the default values. This lock is acquired by our threads just before the count value is incremented. This ensures that the count increment operation is atomic, which means that it either executes completely, or does not execute at all. Now when we run this modified code, we can see that the results are consistent.
<img src="{{ site.baseurl }}/assets/images/5_3_finalres.jpg" alt="disassembly">

<h4><u>References and further reading</u></h4>
1. [Operating systems - Three easy pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/)
2. [Jacob Sorber's playlist on threads](https://www.youtube.com/playlist?list=PL9IEJIKnBJjFZxuqyJ9JqVYmuFZHr7CFM)
