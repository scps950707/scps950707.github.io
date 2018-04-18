---
title: OSDI-Process-Management
date: 2018-04-19 03:07:45
tags:
- NCTU
- OSDI
---
# Process Management

###### tags: `NCTU` `OSDI`

## Traps kernel

CPU提供一些指令EX.interrupt,exception
使用者可以透過軟體中斷做一些處理

x86而言是透過software interrupt:int 0x80會將eax塞入對應的服務編號讓handler找到對應的function

![](https://i.imgur.com/l5UZET5.png =500x)

### x86	Interrupts and Linux Usage

![](https://i.imgur.com/cQoSh0U.png =600x)

## Interrupts and exceptions
參考intel manual
The processor provides two mechanisms for interrupting program execution, interrupts and exceptions:
- interrupt: an asynchronous event that is typically triggered by an I/O device.
- exception:a synchronous event that is generated when the processor detects one or more predefined conditions while executing an instruction. The IA-32 architecture specifies three classes of exceptions: faults, traps, and aborts.
    - A trap is an exception that is reported immediately following the execution of the trapping instruction. Traps allow execution of a program or task to be continued without loss of program continuity. The return address for the trap handler points to the instruction to be executed after the trapping instruction.
    - A fault is an exception that can generally be corrected and that, once corrected, allows the program to be restarted with no loss of continuity. When a fault is reported, the processor restores the machine state to the state prior to the beginning of execution of the faulting instruction. The return address (saved contents of the CS and EIP registers) for the fault handler points to the faulting instruction, rather than to the instruction following the faulting instruction.
    - An abort is an exception that does not always report the precise location of the instruction causing the exception and does not allow a restart of the program or task that caused the exception. Aborts are used to report severe errors, such as hardware errors and inconsistent or illegal values in system tables.
- [What is the difference between Trap and Interrupt?](https://stackoverflow.com/questions/3149175/what-is-the-difference-between-trap-and-interrupt)



## System call

![](https://i.imgur.com/loBEQqc.png =500x)

![](https://i.imgur.com/QIFcncq.png =700x)

CPU會先跳exception之後再跳system call

- [Linux 一次進程的執行過程及一個ELF文件頭問題](http://monkee.esy.es/?p=1349)

切換的同時stack pointer會改指到kernel stack

## Kernel space vs user space

![](https://i.imgur.com/eJyFpt6.png =500x)

[Anatomy of a Program in Memory](https://manybutfinite.com/post/anatomy-of-a-program-in-memory/)


## Program vs. Process

![](https://i.imgur.com/OJHwsBr.png =550x)

### Process creation and destroy in Linux
- Process creation flow
    - fork
        - creates a child process that is a copy of the current task
        - differs from the parent only in its PID, its PPID, and certain resources
        - copy on write
    - exec
        - loads a new executable into the address space and begins executing it
        ![](https://i.imgur.com/aLj5oGI.png =500x)
    - fork()+exec()=spawn()
- User process/thread
    - create by parent
    - run alternatively in Kernel Mode and in User Mode
    - run kernel function through system calls
- Kernel process/thread
    * normally create while OS booting
    * run only in Kernel Mode
    * run specific kernel function
    * use only linear addresses (will be discussed later in MM)
    * process 0: init (executed during start_kernel())

## Process descriptor

![](https://i.imgur.com/wubaukl.png =150x)

- Where Linux store the process descriptor?
    - Of course memory
    - In kernel space 把一塊(4k)拿來放process descriptor
    ![](https://i.imgur.com/3rVRmWf.png =450x)
    - Per process (lightweight process)
    - Allocated by slab allocator (will be described in MM) – why ?
    - Co‐located with process kernel stack
    - What is process kernel stack?
    A stack space used while process is running at kernel mode
    - Why co‐located with process kernel stack?
    Easy to access by stack pointer
- How these process descriptors are managed
    - Who & when create process descriptor?
        - "Parent" creates children & process descriptor
        - While (more precisely should be "before") the process is created
    - Who & when destroy process descriptor?
        - People destroy themselves, "Parent" or "ancestor" destroy the process descriptor
        - While (more precisely should be “after”) the process is terminated
    - Who & when use process descriptor?
        - Scheduler
        - While scheduler is invoked

## Process schedule and context switching in Linux
- Scheduling
    - Find the next suitable process to run
- Context switch
    - Store the context of the current process, restore the context of the next process
- When is the scheduler be invoked
    * Direct invocation vs. Lazy invocation
    * When returning to user‐space from a system call
    * When returning to user‐space from an interrupt handler
    * When an interrupt handler exits, before returning to kernel‐space
    * If a task in the kernel explicitly calls schedule()
    * If a task in the kernel blocks (which results in a call to schedule())
    * Timer interrupt 通常linux早期都是每秒1000次
    * tick kernel很耗電 現代OS已經不太採用

### Timer interrupt basics
![](https://i.imgur.com/qWvNsAy.png =300x)


## Context
### Process context

進kernel space不代表真的有context "switch"僅是mode的轉換

![](https://i.imgur.com/2psSnDl.png =600x)

### Interrupt Context
遇到interrupt會保存當下部份的state(register等等)再進入interrupt context(overhead不會像context switch)那麼大

![](https://i.imgur.com/tFwaVDE.png =600x)

kernel space內能不能interrupt?
效率 vs 即時反應的取捨

### Example Flow
![](https://i.imgur.com/8vz90Pw.png =600x)


## Scheduler呼叫時機

- When returning to user-space from a system call
EX:讀檔會需要等待,所以process自己設成idle 然後call scheduler
![](https://i.imgur.com/ndqKXA2.png =350x)
- If a task in the kernel blocks (which results in a call to schedule())
EX:call一個system call本身需要等待被lock住的kernel resource
![](https://i.imgur.com/nn5P62n.png =350x)
- When returning to user‐space from an interrupt handler
![](https://i.imgur.com/JjczRqJ.png =350x)
- When an interrupt handler exits, before returning to kernel‐space
![](https://i.imgur.com/pz4gtz7.png =350x)
- If a task in the kernel explicitly calls schedule()
![](https://i.imgur.com/PIVLbd9.png =350x)

## Preemption
### User preemption
Occurs when the kernel is in a safe state and about to return to user‐space
![](https://i.imgur.com/B8IyFWE.png =600x)

### Kernel preemption
Linux kernel is possible to preempt a task at any point, so long as the kernel does not hold a lock
![](https://i.imgur.com/rSgurmg.png =600x)

### Preemptive Kernel
* Non‐preemptive kernel supports user preemption
* Preemptive kernel supports kernel/user preemption
* Kernel can be interrupted != kernel is preemptive
    * Non‐preemptive kernel, interrupt returns to interrupted process
    * Preemptive kernel, interrupt returns to any schedulable process
* 2.4 is a non‐preemptive kernel
* 2.6 is a preemptive kernel
* 2.6 could disable CONFIG_PREEMPT

![](https://i.imgur.com/YMkaon9.png =600x)
在被lock的過程中不能被preemption

![](https://i.imgur.com/FGNYccX.png =600x)
沒被lock的情況下可以但是做完其他task必須回kernel繼續

## Single Core vs. Multi‐core
為何CPU可以同時執行同一段code但不能同時讀取同一塊memory?
Ans:因為CPU有instruction cache
![](https://i.imgur.com/eBAArLD.png =600x)

## Linux scheduler example
![](https://i.imgur.com/1FEEMp8.png =600x)
- Timeslice function
    | Type of Task      | Nice Value | Timeslice Duration    |
    |-------------------|------------|-----------------------|
    | Initially created | parent's   | half of parent's      |
    | Minimum Priority  | +19        | 5ms(MIN_TIMESLICE)    |
    | Default Priority  | 0          | 100ms(DEF_TIMESLICE ) |
    | Maximum Priority  | -20        | 800ms(MAX_TIMESLICE)  |

    ![](https://i.imgur.com/jmmsvjO.png =500x)

## Process schedule and context switching in Linux
- Priority‐based scheduler
- Dynamic priority‐based scheduling
    - Dynamic priority
        - Normal process
            - nice value: ‐20 to +19 (larger nice values imply you are being nice to others)
- Static priority
    - Real‐time process
        - 0 to 99
- Total priority: 140

### Linux O(1) cheduler
CPU利用find leading bit指令去找table中要schedule的task
原則就是先把priority高,有timeslice並且為running state的task做完
![](https://i.imgur.com/AJx63H7.png =600x)

### Context switch
* Context switch
    * Hardware context switch
        * Task State Segment Descriptor (Old Linux)
    * Step by step context switch
        * Better control and optimize
* Context switch
    * switch_mm()
        * Switch virtual memory mapping
    * switch_to()
        * Switch processor state
* Process switching occurs only in kernel mode
* The contents of all registers used by a process in User Mode have already been saved


#### Hardware context switch
![](https://i.imgur.com/oUltFsi.png =600x)

How x86 helps in Context Switch
![](https://i.imgur.com/kiCRp96.png =600x)

- [Context Switching on x86](http://samwho.co.uk/blog/2013/06/01/context-switching-on-x86/)
- [Context Switching](https://wiki.osdev.org/Context_Switching)
