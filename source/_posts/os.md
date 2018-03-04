---
title: Operating System Note
date: 2018-02-01 01:35:25
tags: note
---
# Operating System

## Thread
### Property[^1]
- Light-weighted threads switch between threads but not between virtual memories
- Threads allow user programs to continue after starting I/O
- Each thread can run in parallel on a different processor
- Extensive sharing makes CPU switching among peer threads and creation of threads inexpensive compared to processes
- Thread context switch still requires
    - Register set switch
    - No memory management related work!!!

[^1]:[Threads](http://www.personal.kent.edu/~rmuhamma/OpSystems/Myos/threads.htm)

### Compare to process
| Per process items           | Per thread items |
|-----------------------------|------------------|
| Address space               | Program counter  |
| Global variables            | Registers        |
| Open files                  | Stack            |
| Child processes             | State            |
| Pending alarms              |                  |
| Signals and signal handlers |                  |
| Accounting information      |                  |

![](https://i.imgur.com/DWrW65b.png =297x332)


### User-level Threads(pthread)
User-level threads implement in user-level libraries, rather than via systems calls, so thread switching does not need to call operating system and to cause interrupt to the kernel. In fact, the kernel knows nothing about user-level threads and manages them as if they were single-threaded processes.
![](https://i.imgur.com/4CTWgGJ.png =350x300)
#### Advantages
- User-level threads does not require modification to operating systems.
- Each thread is represented simply by a PC, registers, stack and a small control block, all stored in the user process address space.
- The procedure that saves the local thread state and the scheduler are local procedures, hence no trap to kernel, no context switch, no memory switch, and this makes the thread scheduling very fast.
#### Disadvantages
- There is a lack of coordination between threads and operating system kernel. Therefore, process as whole gets one time slice irrespect of whether process has one thread or 1000 threads within. It is up to each thread to relinquish control to other threads.
- User-level threads requires non-blocking systems call i.e., a multithreaded kernel. Otherwise, entire process will blocked in the kernel, even if there are runable threads left in the processes. For example, if one thread causes a page fault, the process blocks.

### Kernel-Level Threads
In this method, the kernel knows about and manages the threads. No runtime system is needed in this case. Instead of thread table in each process, the kernel has a thread table that keeps track of all threads in the system. In addition, the kernel also maintains the traditional process table to keep track of processes. Operating Systems kernel provides system call to create and manage threads.
![](https://i.imgur.com/D9fWuTA.png =300x300)
#### Advantages
- Because kernel has full knowledge of all threads, Scheduler may decide to give more time to a process having large number of threads than process having small number of threads.
- Kernel-level threads are especially good for applications that frequently block.
#### Disadvantages
- The kernel-level threads are slow and inefficient. For instance, threads operations are hundreds of times slower than that of user-level threads.
- Since kernel must manage and schedule threads as well as processes. It require a full thread control block (TCB) for each thread to maintain information

### Multi-threading Models
#### Many-to-One Model
many user threads are mapped to one kernel thread
##### Advantage
- thread management is done in user space, so it is efficient
##### Disadvantage
- Entire process will block if a thread makes a blocking call to the kernel
- Because only one thread can access kernel at a time, no parallelism on multiprocessors is possible
#### One-to-One Model
one user thread maps to kernel thread
##### Advantage
- more concurrency than in many-to-one model
- Multiple threads can run in parallel on multi-processors
##### Disadvantage
- Creating a user thread requires creating the corresponding kernel thread. There is an overhead related with creating kernel thread which can be burden on the performance.
#### Many-to-Many Model
many user threads are multiplexed onto a smaller or equal set  of kernel threads.
##### Advantage
- Application can create as many user threads as wanted
- Kernel threads run in parallel on multiprocessors
- When a thread blocks, another thread can still run


### Linux implementations of POSIX threads[^2]
[^2]:[pthread(7)](http://man7.org/linux/man-pages/man7/pthreads.7.html)

Over time, two threading implementations have been provided by the GNU C library on Linux:
#### LinuxThreads
This is the original Pthreads implementation. Since glibc 2.4, this implementation is no longer supported.
#### NPTL (Native POSIX Threads Library)
This is the modern Pthreads implementation.  By comparison with LinuxThreads, NPTL provides closer conformance to the requirements of the POSIX.1 specification and better performance when creating large numbers of threads.  NPTL is available since glibc 2.3.2, and requires features that are present in the Linux 2.6 kernel.

Both of these are so-called 1:1 implementations, meaning that each thread maps to a kernel scheduling entity.  Both threading implementations employ the Linux **clone** system call. In NPTL, thread synchronization primitives (mutexes, thread joining, and so on) are implemented using the Linux **futex** system call.


## Semaphore
Consider a Semaphore S,the following operations done atomically(hardware support)
### DOWN(P,wait,sleep)

```c
if(S.value>0)
    S.value=S.value-1;
else
    Add process to queue(sleep,doesn't finish DOWN operation)
```
### UP(V,wakeup,signal)
```c
S.value=S.value+1;
if(one or more process were sleeping at that semaphore)
    wakeup process from queue,the process finish DOWN operation
```
*Thus, after an an up on a semaphore with process sleeping on it, the semaphore will still be 0,but there will be one fewer process sleeping on it*

### Implementation[^3]
[^3]:[Semaphore Implementation](http://pages.cs.wisc.edu/~bart/537/lecturenotes/s10.html)
#### Uniprocessor solution
```c
class semaphore{
    private int count;
    public semaphore (int init)
    {
        count = init;
    }
    public void P ()
    {
        while (1)
        {
            Disable interrupts;
            if (count > 0)
            {
                count--;
                Enable interrupts;
                return;
            }
            else
            {
                Enable interrupts;
            }
        }
    }
    public void V()
    {
        Disable interrupts;
        count++;
        Enable interrupts;
    }
}
```
#### Multiprocessor solution
```c
class semaphore
{
    private int flag;
    private int count;
    private queue q;

    public semaphore( int init )
    {
        flag = 0;
        count = init;
        q = new queue();
    }

    public void P()
    {
        Disable interrupts;
        while ( TAS( flag ) != 0 )
        {
            /* just spin */
        };
        if ( count > 0 )
        {
            count--;
            flag = 0;
            Enable interrupts;
            return;
        }
        Add process to q;
        flag = 0;
        Enable interrupts;
        Redispatch;
    }

    public V()
    {
        Disable interrupts;
        while ( TAS( flag ) != 0 )
        {
            /* just spin */
        };
        if ( q == empty )
        {
            count++;
        }
        else
        {
            Remove first process from q and wake it up;
        }
        flag = 0;
        Enable interrupts;
    }
}
```
:::info
The vendor of multi-core cpus has to take care that the different cores coordinate themselves when executing instructions which guarantee atomic memory access.[^4]
:::

## Notes
- [UNIVERSITY OF WISCONSIN-MADISON Computer Sciences Department](http://pages.cs.wisc.edu/~bart/537/lecturenotes/)
- [Department of Computer Science, Kent State University](http://www.personal.kent.edu/~rmuhamma/OpSystems/os.html)
- [Operating System Study Guide](http://www.csie.ntnu.edu.tw/~swanky/os/)


[^4]:[Critical sections with multicore processors](http://stackoverflow.com/a/982797)
