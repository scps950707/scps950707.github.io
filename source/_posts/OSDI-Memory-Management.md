---
title: OSDI-Memory Management
date: 2018-05-13 17:50:37
tags:
- OSDI
- NCTU
---
# Memory Management

###### tags: `NCTU` `OSDI`

## Memory access in a program

```clike
int main()
{
	int a = 5;
	int b = a + 6;
	return 0;
}
```

![](https://i.imgur.com/UJmcIn5.png =450x)

![](https://i.imgur.com/iqOyFv1.png =450x)
左側的是code segment右側的是data segment

![](https://i.imgur.com/cfNWu3X.png =450x)
Physical memory的方法

## Physical and virtual address
### Physical Memory vs. Virtual Memory
- Why not physical memory?
- Benefits for using virtual memory
    - Physical memory shared by multiple processes
    - Isolate processes from each other
    - Large/contiguous per process address space
    - Use memory more efficiently
        ex:Pyscial memory執行時要將所有program片段讀到memory但是virtual memory不需要在branch很多的program較有優勢
- Disadvantage for using virtual memory
    - Expensive and slow
- What kind of memory will you use if we only have single process?A:關掉MMU
- or you are the only developer of all processes, or you need very fast memory accesses?A:Physical

### Virtual addresses
* The memory addresses users and compiler
* believed
* The code/data are stored in physical memory
* The virtual memory addresses are translated into physical memory addresses (by CPU) and then CPU can access the code/data
* How can a CPU translate the addresses?
    * Offset? Table lookup?

![](https://i.imgur.com/I5GwDvj.png =500x)

instruction和data的virtual address本身都會透過CPU上的MMU去做轉換

## Segmentation

![](https://i.imgur.com/9FVXMbY.png =550x)
![](https://i.imgur.com/pFeDY1w.png =500x)

- 轉換的table可以放
    - memory:貴 要讀取兩次 所以通常有限制大小 
    - CPU:快 可以放的不多
- CPU會檢查d(offset)會不會超過limit,產生相對應的trap

## Paging
![](https://i.imgur.com/cRGdHQ6.png =500x)

With TLB
![](https://i.imgur.com/UH55MEM.png =500x)

### Page Table and Virtual Address
![](https://i.imgur.com/G80BBxa.png)
如果要的address在TLB內的valid=0(沒有真的在memory內)就會觸發page fault,對應的handler會去memory找一塊physical memory並把這塊的address填到TLB內

### TLB Flush When Context Switch
![](https://i.imgur.com/JzPvWwq.png =500x)
通常要清掉TLB但是也可以設計成不要清掉

Q:利用TLB with PID info的技巧可以避免flush,那麼CPU multi-core時每個core有自己的TLB,kernel會不會嘗試儘量讓同個process排到同一個之前跑的core(當下可能是busy所以照樣等待)來避免switch到一個完全沒有先前TLB的core?
A:linux會抉擇,會讓process有TLB的狀況下跑更快,直到TLB剩餘的加速能力比拿到新的core全部TLB miss後重新填回來還不划算時就會切換到新的core

Is the TLB shared between multiple cores?[^1]


### Memory Cache
- Logical
    ![](https://i.imgur.com/DtYhajk.png =500x)
    - 讀0f50在logical cache中hit快速得到資料,此處為push指令
    - 沒有hit就往TLB找
    - Pid or flush while context switch
- Physical
    ![](https://i.imgur.com/uktAtqn.png =500x)
    - Slow but share without flush

![](https://i.imgur.com/wUJcEtp.png)
架構轉換原因之一為早期系統沒有很多process所以context switch沒那麼多

### Segmentation with paging (x86 example)
![](https://pdos.csail.mit.edu/6.828/2009/lec/x86_translation.svg =600x)
透過selector去table中選擇一個descripotor取得base和offset之後算出linear address在透過paging算出真正的physical address
![](https://i.imgur.com/oJIPA4W.png)
kernel可以看到全域的GDT的kernel code,data user透過LDT看到自己的code,data

### User address space
- 4G for 32 bits processor
- How about kernel?
    - Kernel uses another 4G address space ?
    - Context switch for system calls?
- How about user and kernel share the same 4G spaces?
    - User program can use X
    - Kernel program can use 4G‐X
- Can user accesses the kernel data?
    - Limited by user data segment
- Can kernel access the user data?
    - Kernel data segment covers 4G
![](https://i.imgur.com/qTmEScL.png =500x)

## Physical memory management
![](https://i.imgur.com/83Ow9Fk.png =500x)
- Always required
    - OS has to know how physically memory is used
- Kernel (physical contiguous and logical contiguous addresses are preferred)
    - For better performance
- Without MMU
    - Low cost
    - Better performance(不需要TLB 就不需要flush overhead)
    - Deterministic performance(沒有virtual memory跟swapping space這樣access memory的時間較固定)


## How to Protect Memory Access without MMU?
- MPU (Memory Protection Units)
    - Low cost solution for multitasking and memory access protection
    - 在CPU內部維護segment table

    ![](https://i.imgur.com/CAt79Wg.png =500x)


- Region Attributes
    ![](https://i.imgur.com/OGfkqDG.png =400x)
- Example 1
    ![](https://i.imgur.com/dtr4Q7X.png =400x)
- Example 2
    ![](https://i.imgur.com/p7nHPI3.png =400x)
    ![](https://i.imgur.com/4cYWols.png =400x)
- What are the issues for physical memory management?
    - External fragmentation
    - Internal fragmentation
    - Search for empty space

## Fragmentation
### External
![](https://i.imgur.com/eAY5tUr.png =500x)
- 如果用Memory packing去解決,在系統層面cost會太大
- uCLinux Design[^2]
    - Standard Linux allocator: allocates blocks of 2n size.
        - If 65 KB are requested, 128 KB will be reserved, and the remaining 63 KB won’t be reusable in Linux.
    - uClinux 2.4 memory allocator: kmalloc2 (aka page_alloc2)
        - Allocates blocks of 2n size until 4KB
        - Uses 4KB pages for greater requests
        - Stores amounts not greater than 8KB on the start of the memory, and larger ones at the end. Reduces fragmentation
        - Not available yet for Linux 2.6.
        ![](https://i.imgur.com/I1IMaPq.png =150x)

- Frame Table
    - Divide physical memory into frames
        – frame size = page size (why?)
        – Different frame size (why?)
    ![](https://i.imgur.com/ZLA1DjJ.png =300x)

### Internal
當分配的size比frame size小時容易發生

- Frame size跟table size本身是tradeoff
- Different frame size (why?) 在嵌入式系統可使用此技巧,藉由runtime分析之後去做最佳分配

## Kernel Memory Management
### Requirements
- Managing memory have to spend memory
    - Balance between management overhead and waste
- Prefer physical contiguous and logical contiguous addresses
    - For better performance
    - Different from application memory management (logical contiguous but not necessary physical contiguous)
        - Memory utilization is more important
- Avoid external fragmentation
    - Separate large and small allocations
- Avoid internal fragmentation
    - Multiplexing more memory block requests into one page
- Support both large/small blocks allocations and free
    - Large memory block such as DMA
    - Small memory block such as task structure
- Lower empty space search complexities
    - O(1)
- Provide APIs for kernel memory allocation/free
    - For drivers, OS codes
- Provides APIs for realizing application memory allocation/free
    - Allocate logical contiguous but not necessary physical contiguous memory

### Overall Architecture
![](https://i.imgur.com/ptYo8Sm.png =400x)
Buddy System會去要實際的physical page,Slab Allocator目的是解決internal fragmentaion,然後最後才會mapping成user space看的VMA


### System Components around Slab Allocaters
![](https://i.imgur.com/HUSVUOS.png =500x)
Slab Allocator本身也是呼叫Page Allocator

### Linux Memory Zones
![](https://i.imgur.com/neHylkt.png =400x)
通常分法
high:user
normal:kernel
Relationship Between Nodes, Zones and Pages
![](https://i.imgur.com/Q8CHgCb.png =500x)
strcut page對到physical frame

### Linux Memory management
1. Buddy allocation
    - Why? O(1)
    - 適合大塊記憶體 但是internal fragmentation嚴重
    ![](https://i.imgur.com/8Y2EihT.png)
    Zoned Buddy Allocator
    ![](https://i.imgur.com/AXZeHiy.png)
1. Slab allocation
    - The Slab Allocator: An Object-Caching Kernel Memory Allocator[^3]
    - Slab allocators in the Linux Kernel[^4]
    - 讓好幾筆allocation用同個page
    - 讓kernel program不用直接面對page allocator
    - 把常alloc/free的object(structure) cache下來
    - 避免每個kernel structure擺在太近造成cache line一直刷新
    - 適合小塊記憶體

        ![](https://i.imgur.com/mOvoD8K.png =400x)
    cache:同一類型的object
    slabs:連續的page組成page中放相同類型object
    - Page to Cache and Slab Relationship
        ![](https://i.imgur.com/pTC0NNp.png =300x)
    - Slab With Descriptor On‐Slab
        ![](https://i.imgur.com/ffhF1jl.png =450x)
    - [slab.c](https://github.com/torvalds/linux/blob/master/mm/slab.c#L37)
    - In order to reduce fragmentation, the slabs are sorted in 3 groups:
        - full slabs with 0 free objects
        - partial slabs
        - empty slabs with no allocated objects
    - If partial slabs exist, then new allocations come from these slabs, otherwise from empty slabs or new slabs are allocated.


1. kmalloc
    - similar to that of user‐space's familiar malloc() routine
    - byte‐sized chunks
    - memory allocated is physically contiguous
    - Through slab allocator
1. vmalloc
    - virtually contiguous and not necessarily physically contiguous
    - user‐space allocation function works
    - allocating potentially non-contiguous chunks of physical memory and "fixing up" the page tables to map the memory into a contiguous chunk of the logical address space

## Process address space
- How The Kernel Manages Your Memory[^6]
- Virtual Addresses Before/After Context Switches
![](https://i.imgur.com/UZBp1Zn.png =500x)
- Per Process Address Space
![](https://i.imgur.com/RlVSqMN.png =200x250)
- task_struct and mm_struct
![](https://i.imgur.com/8A7iN3b.png =500x)
- VMA
當使用library時可能會觸發page fault來把dll讀進memory
![](https://i.imgur.com/RO5crlF.png =500x)
- Memory Access
![](https://i.imgur.com/ORxnrGH.png =400x)
- Page Fault
![](https://i.imgur.com/PhaXlU7.png =400x)
![](https://i.imgur.com/w0O7N7l.png =400x)
### Linux Kernel Thread and mm_struct
- Kernel threads do not have a process address space
    - mm field of a kernel thread's process descriptor is NULL
- Lack of an address space is fine, because kernel threads do not ever access any user‐space memory
- Better performance

## Management Regions
- mm_struct, vm_area_struct, and page
![](https://i.imgur.com/CI55s0g.png)
- VMA, Page and Frame
![](https://i.imgur.com/Od9JvJM.png)
- Memory Region
![](https://i.imgur.com/TG6QBOm.png =500x)
- Red-Black Tree
    - O(logn)
    - Compare to AVL tree[^5]
        - Search slower
        - Insert and delete faster
    
    ![](https://i.imgur.com/b8oi6Se.png)
- Enlarge VMA
![](https://i.imgur.com/UFMsgsv.png =450x)

## Memory mapping

![](https://i.imgur.com/MUWU28r.png)
- File mapping and memory layout
![](https://i.imgur.com/X0EB8Ua.png)

- Data Structure for File Memory Mapping
![](https://i.imgur.com/7EBoA0P.png)

### Two Types of Memory Mapping
- A file mapping maps a memory region to a region of a file
    - backing store = file
    - as long as the mapping is established, the content of the file can be read from or written to using direct memory access (“as if they were variables”)
- An anonymous mappings maps a memory region to a fresh “virtual” memory area filled with 0
    - backing store = zero‐ed memory area
### Having memory mapped pages in common
- Thanks to virtual memory management, different processes can have mapped pages in common
- More precisely, mapped pages in different processes can refer to physical memory pages that have the same backing store
- That can happen in two ways:
    - through fork, as memory mappings are inherited by children
    - when multiple processes map the same region of a file
### Shared vs private mappings
- With mapped pages in common, the involved processes might see changes performed by others to mapped pages in common, depending on whether the mapping is:
    - private mapping in this case modifications are not visible to other processes.
        - pages are initially the same, but modification are not shared, as it happens with copy‐on‐write memory after fork
        - private mappings are also known as copy‐on‐write mappings
    - shared mapping in this case modifications to mapped pages in common are visible to all involved processes
        - pages are not copied‐on‐write
## Shared file mapping
- Effects
    - processes mapping the same region of a file share physical memory frames
        - more precisely: they have virtual memory pages that map to the same physical memory frames
    -  additionally, the involved physical frames have the mapped file as ultimate backing store
        - modifications to the (shared) physical frames are saved to the mapped file on disk

- Use cases
    - memory‐mapped I/O, as an alternative to read/write
        - as in the case of private file mapping, but here it works for both reading and writing data
    - Inter‐process communication, with the following characteristics
        - data‐transfer (not byte stream)
        - with filesystem persistence
        - among unrelated processes
![](https://i.imgur.com/gPWCaMy.png)

## Memory-mapped I/O
- Given that
    - memory content is initialized from file
    - changes to memory are reflected to file
- We can perform I/O by simply changing bytes of memory.
- Access to file mappings is less intuitive than sequential read/write operations
    - the mental model is that of working on your data as a huge byte array (which is what memory is, after all)
    - a best practice to follow is that of defining struct‐s that correspond to elements stored in the mapping, and copy them around with memcpy & co

### Advantages
- performance gain: 1 memory copy
    with read/write I/O each action involves 2 memory copies:
    1. between user‐space and kernel buffers
    2. between kernel and memory(buffer) of disk
- buffers and the I/O device
    - with memory‐mapped I/O only the 2nd copy remains
    - flash exercise: how many copies for standard I/O?
- performance gain: no context switch
    - no syscall and no context switch is involved in accessing mapped memory
    - page faults are possible, though reduced memory usage
    - we avoid user‐space buffers ! less memory needed
    - if memory mapped region is shared, we use only one set of
- buffers for all processes seeking is simplified
    - no need of explicit lseek, just pointer manipulation

### Disadvantages
- memory garbage
    - the size of mapped regions is a multiple of system page size
    - mapping regions which are way smaller than that can result in a significant waste of memory
- memory mapping must fit in the process address space
    - on 32 bits systems, a large number of mappings of various sizes might result in memory fragmentation
    - it then becomes harder to find continuous space to grant large memory mappings
    - the problem is substantially diminished on 64 bits systems
- there is kernel overhead in maintaining mappings
    - for small mappings, the overhead can dominate the advantages
    - memory mapped I/O is best used with large files and random access

## Page Cache
![](https://i.imgur.com/Cz4auag.png =500x)

![](https://i.imgur.com/W6ATsgx.png)


---
- [Main Memory CS 4410, Operating Systems Fall 2016 Cornell University](http://www.cs.cornell.edu/courses/cs4410/2016fa/slides/13-memory.pdf)
- [W4118: Linux memory management](http://www.cs.columbia.edu/~junfeng/13fa-w4118/lectures/l20-adv-mm.pdf)
- [Programmation Systèmes Memory Mapping, Stefano Zacchiroli](https://upsilon.cc/~zack/teaching/1112/progsyst/cours-09.pdf)

[^1]:[Is the TLB shared between multiple cores?](https://stackoverflow.com/questions/34437371/is-the-tlb-shared-between-multiple-cores)
[^2]:[Introduction to uClinux](https://bootlin.com/doc/legacy/uclinux/uclinux_introduction.pdf)
[^3]:[The Slab Allocator: An Object-Caching Kernel Memory Allocator](https://people.eecs.berkeley.edu/~kubitron/cs194-24/hand-outs/bonwick_slab.pdf)
[^4]:[Slab allocators in the Linux Kernel](https://events.static.linuxfound.org/sites/events/files/slides/slaballocators.pdf)
[^5]:[Red black tree over avl tree](https://stackoverflow.com/a/23277366)
[^6]:[How The Kernel Manages Your Memory](https://manybutfinite.com/post/how-the-kernel-manages-your-memory/)
