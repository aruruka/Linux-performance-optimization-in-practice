# 15 Basics: How does Linux memory work?

In the previous sections, we have learned the performance principles and optimization
methods of CPU together. Next, we will enter another section-memory.

Like CPU management, memory management is one of the core functions of the
operating system. Memory is mainly used to store system and application instructions,
data, cache, etc.

So, how does Linux manage memory? Today, I will take you to take a look at this question.

## Memory mapping

Speaking of memory, can you tell me how much memory you have on this computer?
I guess you remember it clearly, because this is an important parameter we consider
first when we buy, for example, my laptop memory is 8GB.

The memory capacity we usually refer to, like the 8GB I just mentioned, actually
refers to physical memory. Physical memory is also called main memory, and the
main memory used by most computers is dynamic random access memory (DRAM). Only
the kernel can directly access physical memory. So, what should a process do when
it wants to access memory?

The Linux kernel provides each process with an independent virtual address space,
and this address space is continuous. In this way, the process can easily access
memory, more precisely, access virtual memory.

The interior of the virtual address space is divided into **kernel space and user
space**. Processors with different bit length (that is, the maximum length of
data that can be processed by a single CPU instruction) have different address
space ranges. For example, for the most common 32-bit and 64-bit systems, I drew
two pictures to represent their virtual address spaces, as follows:

<center><a href="https://ibb.co/jJcSgXD">
    <img src="https://i.ibb.co/nPHFzJj/Virtual-Memory-And-Bit-Length.png" alt="Virtual-Memory-And-Bit-Length" border="0">
</a></center>

It can be seen from this that the kernel space of the 32-bit system occupies 1G,
which is at the highest point, and the remaining 3G is user space. The kernel
space and user space of a 64-bit system are both 128T, occupying the highest and
lowest parts of the entire memory space respectively, and the remaining middle
part is undefined.

Remember the user mode and kernel mode of the process? When a process is in user
mode, it can only access user space memory; only after entering kernel mode can
it access kernel space memory. Although the address space of each process includes
the kernel space, these kernel spaces are actually associated with the same physical
memory. In this way, after the process switches to the kernel state, it can
easily access the kernel space memory.

Since each process has such a large address space, the sum of the virtual memory
of all processes is naturally much larger than the actual physical memory.
Therefore, not all virtual memory will allocate physical memory, only those virtual
memory actually used will allocate physical memory, and the allocated physical
memory is managed through **memory mapping**.

Memory mapping is actually mapping **virtual memory addresses** to **physical
memory addresses**. In order to complete memory mapping, the kernel maintains a
page table for each process, recording the mapping relationship between virtual
addresses and physical addresses, as shown in the following figure:

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/R4Mb908/Page-Table-Of-Memory-Mapping.png" alt="Page-Table-Of-Memory-Mapping" border="0">
</a></center>

The page table is actually stored in the CPU's memory management unit MMU, so that
under normal circumstances, the processor can directly find out the memory to be
accessed through the hardware.

When the virtual address accessed by the process cannot be found in the page table,
the system will generate a page fault exception, enter the kernel space to allocate
physical memory, update the process page table, and finally reclaim to the user space
to resume the operation of the process.

In addition, I mentioned in 🔗[the article on CPU context switching](https://time.geekbang.org/column/article/69859) that
TLB (Translation Lookaside Buffer) will affect the
memory access performance of the CPU, which can actually be explained here.

The TLB is actually a cache of page tables in the MMU. Since the virtual address
space of the process is independent, and the access speed of the TLB is much faster
than that of the MMU, by reducing the context switching of the process and the
number of refreshes of the TLB, the utilization rate of the TLB cache can be
increased, and the CPU performance can be improved and hence improve memory access
performance.

However, it should be noted that the MMU does not manage memory in units of bytes,
but specifies a minimum unit of memory mapping, that is, a page, usually 4 KB in
size. In this way, each memory mapping needs to be associated with a memory space
of 4 KB or an integer multiple of 4KB.

Another problem caused by the page size being only 4 KB is that the entire page
table can become very large. For example, a 32-bit system alone requires more
than 1 million page table entries (4GB/4KB) to map the entire address space. In
order to solve the problem of too many page table entries, Linux provides two
mechanisms, namely multi-level page table and large page (HugePage).

The multi-level page table is to divide the memory into blocks for management,
and change the original mapping relationship to the block index and the offset
within the block. Since the virtual memory space usually only uses a small part,
the multi-level page table only saves these blocks in use, which can greatly
reduce the number of entries in the page table.

Linux uses a four-level page table to manage memory pages. As shown in the figure
below, the virtual address is divided into 5 parts. The first 4 table entries are
used to select pages, and the last index represents the offset within the page.

<center><a href="https://ibb.co/JC5wF1P">
    <img src="https://i.ibb.co/Trmxqf5/Multi-Level-Page-Table.png" alt="Multi-Level-Page-Table" border="0">
</a></center>

Looking at the large page, as the name suggests, it is a memory block that is
larger than a normal page, and the common sizes are 2MB and 1GB. Huge pages are
usually used on processes that use a lot of memory, such as Oracle, DPDK, etc.

Through these mechanisms, under the mapping of the page table, the process can
access the physical memory through the virtual address. So specifically in a
Linux process, how is this memory used?

## Virtual memory space distribution

First of all, we need to further understand the distribution of virtual memory
space. Needless to say, the uppermost kernel space, the user space memory below
is actually divided into multiple different segments. Taking the 32-bit system as
an example, I drew a diagram to show their relationship.

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/YNQC74X/Virtual-Memory-Space.png" alt="Virtual-Memory-Space" border="0">
</a></center>

You can see from this picture that user space memory has five different memory segments from low to high.

1. Read-only segment, including code and constants, etc.
2. Data segment, including global variables, etc.
3. The heap, including dynamically allocated memory, grows upwards from low addresses.
4. The file mapping segment, including dynamic library, shared memory, etc., grows downward from the high address.
5. The stack, including the context of local variables and function calls, etc. The stack size is fixed, typically 8 MB.

Among the five memory segments, the memory of the heap and file mapping segments
is allocated dynamically. For example, using malloc() or mmap() of the C standard
library, you can dynamically allocate memory in the heap and file mapping segments,
respectively.

In fact, the memory distribution of the 64-bit system is similar, but the memory
space is much larger. So, the more important question is, how is the memory allocated?

## Memory allocation and reclaim

malloc() is a memory allocation function provided by the C standard library,
corresponding to the system call, there are two implementations, namely brk() and mmap().

For small blocks of memory (less than 128K), the C standard library uses brk() to
allocate, that is, to allocate memory by moving the position of the top of the heap.
After the memory is released, it will not be reclaimed to the system immediately,
but will be cached so that it can be reused.

For large blocks of memory (greater than 128K), use memory mapping mmap() to
allocate directly, that is, find a piece of free memory in the file mapping segment
and allocate it.

These two methods, of course, have their own advantages and disadvantages.

The brk() approach of caching can reduce the occurrence of page fault exceptions
and improve memory access efficiency. However, since the memory is not reclaimed
to the system, frequent memory allocation and release will cause memory fragmentation
when the memory work is busy.

The memory allocated by mmap() will be directly reclaimed to the system when it is
released, so a page fault will occur every time mmap is executed. When the memory
work is busy, frequent memory allocation will cause a large number of page fault
exceptions, which will increase the management burden of the kernel. This is also
why malloc only uses mmap for large chunks of memory.

After understanding these two calling methods, we need to be clear, that is, when
these two calling methods occur, memory is not actually allocated. These memories
are only allocated when they are accessed for the first time, that is, they enter
the kernel through a page fault exception, and then the kernel allocates the memory.

Overall, Linux uses a buddy system to manage memory allocation. As we mentioned
earlier, these memories are managed in units of pages in the MMU, and the same
is true for the partner system, which manages memory in units of pages, and will
reduce memory fragmentation (such as caused by the brk method) by merging adjacent
pages memory fragmentation.

You may think of a question, if you encounter an object smaller than a page, such
as less than 1K, how to allocate memory?

In the actual system operation, there are indeed a large number of objects that
are smaller than a page. If a separate page is allocated for them, it will be a
waste of memory.

Therefore, in user space, the memory allocated by malloc through brk() is not
reclaimed to the system immediately when it is released, but is cached for reuse.
In kernel space, Linux manages small memory through the slab allocator. You can
think of slab as a cache built on the buddy system. Its main function is to
allocate and release small objects in the kernel.

For memory, if it is only allocated but not released, it will cause a memory leak
and even run out of system memory. Therefore, after the application runs out of
memory, it needs to call free() or unmap() to release the memory not needed.

Of course, the system will not let a process use up all the memory. When the memory
is found to be tight, the system will reclaim the memory through a series of mechanisms,
such as the following three approaches:

- Reclaim the cache, such as using the LRU (Least Recently Used) algorithm to reclaim the least recently used memory pages;
- Reclaim infrequently accessed memory, and write infrequently used memory directly to disk through the swap partition;
- Kill the process. When the memory is tight, the system will directly kill the process that occupies a large amount of memory through OOM (Out of Memory).

Among them, the second method will use the swap partition (hereinafter referred to
as Swap) when reclaiming infrequently accessed memory. Swap actually uses a piece
of disk space as memory. It can store the data temporarily unused by the process
to the disk (this process is called swapping out), and when the process accesses
the memory, it can read the data from the disk into the memory (this process is
called swapping in).

Therefore, you can find that Swap increases the available memory of the system.
However, it should be noted that Swap exchange usually only occurs when there is
insufficient memory. And because the read and write speed of the disk is much slower
than that of the memory, Swap will cause serious memory performance problems.

The OOM (Out of Memory) mentioned in the third way is actually a protection
mechanism of the kernel. It monitors the memory usage of processes and uses
oom_score to score the memory usage of each process:

- The larger the memory consumed by a process, the larger the oom_score;
- The more CPU a process takes to run, the smaller the oom_score.

In this way, the larger the oom_score of the process, the more memory it consumes,
and the easier it is to be killed by OOM, so that the system can be better protected.

Of course, for actual work needs, the administrator can manually set the oom_adj
of the process through the /proc file system, thereby adjusting the oom_score of
the process.

The range of oom_adj is [-17, 15]. The larger the value, the easier the process
is to be killed by OOM; the smaller the value, the less likely the process is to
be killed by OOM, and -17 means that OOM is prohibited.

For example, with the following command, you can reduce the oom_adj of the `sshd`
process to -16, so that the `sshd` process will not be easily killed by OOM.

```shell
echo -16 > /proc/$(pidof sshd)/oom_adj
```

## How to check memory usage

By understanding the distribution of memory space, as well as the allocation and
recovery of memory, I think you should have a general understanding of how memory
works. Of course, the actual working principle of the system is more complicated,
and some other mechanisms will also be involved. Here I only talk about the most
important principles. After mastering these, you can have a main understanding of
the operation of memory, so that you don't only have a pile of terminology in your
mind.

So after understanding the working principle of memory, how can we check the
system memory usage?

In fact, in the previous study of CPU content, we also mentioned some related tools.
Here, the first thing you think of should be the free tool. Here is a sample output
from free:

```shell
# 注意不同版本的free输出可能会有所不同
$ free
              total        used        free      shared  buff/cache   available
Mem:        8169348      263524     6875352         668     1030472     7611064
Swap:             0           0           0
```

You can see that the output of `free` is a table, and the values in it are in
bytes by default. The table has a total of two rows and six columns. These two
rows are the usage of the physical memory Mem and the swap partition Swap, and
the meaning of each column of the six columns is:

- The first column, total is the total memory size;
- The second column, used is the size of the used memory, including shared memory;
- The third column, free is the size of unused memory;
- The fourth column, shared is the size of the shared memory;
- The fifth column, buff/cache is the size of the cache and buffer;
- The last column, available is the amount of memory available to the new process.

Pay special attention here to the available memory available in the last column.
Available not only includes free memory, but also includes reclaimable caches,
so it is generally larger than free memory. However, not all caches can be reclaimed,
as some may be in use.

However, we know that `free` shows the memory usage of the entire system. If you
want to view the memory usage of the process, you can use tools such as `top` or `ps`.
For example, here is a sample output from `top`:

```shell
# 按下M切换到内存排序
$ top
...
KiB Mem :  8169348 total,  6871440 free,   267096 used,  1030812 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7607492 avail Mem


  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
  430 root      19  -1  122360  35588  23748 S   0.0  0.4   0:32.17 systemd-journal
 1075 root      20   0  771860  22744  11368 S   0.0  0.3   0:38.89 snapd
 1048 root      20   0  170904  17292   9488 S   0.0  0.2   0:00.24 networkd-dispat
    1 root      20   0   78020   9156   6644 S   0.0  0.1   0:22.92 systemd
12376 azure     20   0   76632   7456   6420 S   0.0  0.1   0:00.01 systemd
12374 root      20   0  107984   7312   6304 S   0.0  0.1   0:00.00 sshd
...
```

The top of the `top` output interface also shows the overall memory usage of the system.
These data are similar to `free`, so I won't repeat the explanation. Let's look
at the following content, several columns of data related to memory, such as VIRT,
RES, SHR, and %MEM.

These data include the most important memory usage of the process, let's look at
them one by one.

- VIRT is the size of the virtual memory of the process. As long as it is the memory that the process has applied for, even if the physical memory has not been actually allocated, it will be counted.
- RES is the size of resident memory, that is, the size of physical memory actually used by the process, but does not include Swap and shared memory.
- SHR is the size of shared memory, such as shared memory shared with other processes, loaded dynamic link libraries, code segments of programs, etc.
- %MEM is the percentage of physical memory used by the process out of the total system memory.

In addition to knowing these basics, there are two things you should be aware of
when looking at `top` output.

First, virtual memory usually does not allocate all of physical memory. From the
above output, you can find that the virtual memory of each process is much larger
than the resident memory.

Second, the shared memory SHR is not necessarily shared. For example, the code
segment of the program and the non-shared dynamic link library are also counted
in the SHR. Of course, SHR also includes memory that is actually shared between
processes. So when calculating the memory usage of multiple processes, do not
directly add the SHRs of all processes to get the result.

## Summary

Today, we sorted out how Linux memory works. For ordinary processes, what it can
see is actually the virtual memory provided by the kernel. These virtual memories
also need to be mapped to physical memory by the system through the page table.

When a process applies for memory through malloc(), the memory is not allocated
immediately, but is allocated in the kernel through a page fault exception when
it is accessed for the first time.

Since the virtual address space of a process is much larger than the physical memory,
Linux also provides a series of mechanisms to deal with the problem of insufficient
memory, such as cache recovery, swap partition Swap, and OOM.

When you need to know the memory usage of the system or process, you can use `free`
and `top`, `ps` and other performance tools. They are the most commonly used performance
tools when analyzing performance problems. I hope you can use them proficiently
and truly understand the meaning of each indicator.