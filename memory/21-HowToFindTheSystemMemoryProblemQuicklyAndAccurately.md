# 21 Routines: How to find the system memory problem "quickly and accurately"?

In the previous sections, we have analyzed various common memory performance
problems through several case studies. I believe that through them, you have a
basic idea of memory performance analysis and are familiar with many tools for
analyzing memory performance. You must be wondering if there is a way to locate
memory problems quickly. Once the memory bottleneck is located, what are the ideas
to optimize the memory?

Today, I'm going to help you sort out how you can quickly locate system memory
and summarize the related solution ideas.

## Memory profiling metrics Memory Performance Metrics

In order to analyze memory performance bottlenecks, you first need to know how to
measure memory performance, which is the problem of performance metrics. Let's
start by reviewing the memory performance metrics that we learned in the previous
sections.

You can first find a piece of paper yourself and write it from memory; or open
the previous article and summarize it yourself.

The first thing you can most easily think of is system memory usage, such as
used memory, remaining memory, shared memory, available memory, cache and buffer
usage, etc.

- Used memory and remaining memory are easy to understand, they are the memory that has been used and the memory that is not yet used.
- Shared memory is implemented through tmpfs, so its size is also the size of the memory used by tmpfs. tmpfs is actually a special kind of cache.
- Available memory is the maximum memory that can be used by a new process, and it includes both remaining memory and recyclable cache.
- The cache consists of two parts. One part is the page cache of the file read from disk, which is used to cache the data read from disk and can be accessed again faster in the future. The other part, the recyclable memory in the Slab allocator.
- Buffers are temporary storage for raw disk blocks and are used to cache the data that will be written to disk. This allows the kernel to centralize the scattered writes and optimize the disk writes in a uniform manner.

The second category that comes easily to mind would be process memory usage,
such as virtual memory, resident memory, shared memory, and Swap memory of the process.

- Virtual memory, which includes process code segments, data segments, shared memory, heap memory that has been requested, and memory that has been swapped out, etc. Note here that memory that has been requested, even if no physical memory has been allocated yet, counts as virtual memory.
- Resident memory is the physical memory actually used by the process, however, it does not include Swap and shared memory.
- Shared memory, includes both the real shared memory that is shared with other processes, as well as loaded dynamic link libraries and code segments of programs, etc.
- Swap memory is the memory that is swapped out to disk via Swap.

Of course, resident memory in these metrics is generally converted to a percentage
of total system memory, which is the memory usage of the process.

In addition to these indicators that come easily to mind, I would like to highlight
again the missing page exception.

In the principle of memory allocation, I have talked about how the system does
not allocate physical memory for a memory allocation request immediately after
it is invoked, but instead allocates it through a missing page exception when the
request is first accessed. The out-of-page exceptions are further divided into
the following two scenarios.

- When it can be allocated directly from physical memory, it is called a sub-pageout exception.
- When disk I/O intervention (e.g. Swap) is required, it is called a main out-of-page exception.

Obviously, an exceptionally high main page miss means that disk I/O is required,
and then memory accesses will be much slower.

In addition to system memory and process memory, the third important category of
metrics is Swap usage, such as Swap used space, remaining space, swap-in speed
and swap-out speed.

- Used space and remaining space are well understood and are literally the used and unused memory space.
- The swap-in and swap-out speed, on the other hand, indicates the size of the swap-in and swap-out memory per second.

These memory performance indicators need to be memorized and used, and I've put
them together into a mind map that you can save and print out, or summarize for yourself.

<center><a href="https://ibb.co/nbbmcjs">
    <img src="https://i.ibb.co/800s59D/Memory-Profiling-Metrics.png" alt="Memory-Profiling-Metrics" border="0">
</a></center>

## Memory profiling tools

Once we understand the memory performance metrics, we also need to know how we
can get them, that is, we can use performance tools. Here, we use the same approach
to review the various memory performance tools that have been used in the previous
cases. You are still encouraged to recall and summarize them yourself first.

First of all, you should have noticed that all cases use `free`, the most commonly
used memory tool to see the overall memory and swap usage of the system.
Correspondingly, you can use `top` or `ps` to see the memory usage of your processes.

Then, in the principles of cache and buffer chapter, we found the source of the
memory metrics through the proc file system; and dynamically observed the memory
changes through `vmstat`. Compared to `free`, vmstat can distinguish between
cache and buffer, swap-in and swap-out memory sizes, in addition to dynamically
viewing memory changes.

Then, in the cache and buffer case study, we used `cachestat` to see the read
and write hits of the entire system cache and `cachetop` to see the read and
write hits of the cache for each process in order to figure out how the cache is
hitting.

Then, in the case of the memory leak, we used `vmstat`, which found that memory
usage was growing, and `memleak`, which confirmed that a memory leak had occurred.
Using the memory allocation stack given by `memleak`, we found the suspected location
of the memory leak.

Finally, in the case of Swap, we found the buffer and Swap elevation problem using `sar`.
By comparing the remaining memory with the memory threshold of <ins>/proc/zoneinfo</ins>,
we found that the Swap increase was caused by memory reclamation. Finally, we also
identified the processes affected by Swap through the <ins>/proc</ins> file system.

By now, you are once again feeling the "malice" from the world of performance.
Why are there so many performance tools? In fact, as always, it's not hard to
understand how memory works and memorize it in conjunction with performance metrics
to get a handle on how to use the tools.

## Linkage of performance indicators and tools

As with CPU performance analysis, my experience has been with two different dimensions,
organization and memorization.

- Starting with memory metrics makes it easier to correlate the tool with how memory works.
- Starting with performance tools makes it faster to use the tools and find out the performance metrics we want to observe. Especially with limited tools, we have to make full use of every tool at hand to dig out more problems.

Similarly, based on the correspondence between memory performance metrics and tools,
I have made two tables to help you sort out the relationship and understand it.
Of course, you can also use them as "metrics tools" and "tools metrics" guides
to look up when you need them.

The first table, starting with memory metrics, lists which performance tools can
provide those metrics. This way, when actually troubleshooting performance issues,
you know exactly what tools to use to aid in the analysis and provide the metrics
you want.

<center><a href="https://ibb.co/VWBNVCX">
    <img src="https://i.ibb.co/ynsR0kL/Find-Tools-By-Metrics-Memory-Performance.png" alt="Find-Tools-By-Metrics-Memory-Performance" border="0">
</a></center>

The second table, from Performance Tools, organizes the memory metrics that these
common tools can provide. Armed with this table, you can maximize the use of the
tools available and find as many metrics as you want.

You don't have to memorize how to use these tools, you just need to know what
tools are available and the basic metrics they provide. When you really need them,
`man` up and look up their manuals.

<center><a href="https://ibb.co/cYMHK14">
    <img src="https://i.ibb.co/1zh5YLF/Find-Indicator-By-Tool-Cpu.png" alt="Find-Indicator-By-Tool-Cpu" border="0">
</a></center>

## How to quickly analyze memory performance bottlenecks

I'm sure by this point you are familiar with memory performance metrics and know
exactly what tools can be used to get each one.

Does that mean that every time you encounter a memory performance problem, you
have to run all these tools above and then analyze all the memory performance
metrics again?

Naturally not. As we said in the previous CPU performance article, the simple
lookup method, while useful, is also likely to find some potential system bottlenecks.
But the inefficiency and large workload of this method made us reject it in the
first place.

As always, in a real production environment, what we want is to locate system
bottlenecks as **quickly** as possible and then optimize performance as **quickly**
as possible, that is, to solve performance problems quickly and accurately.

So is there any way to analyze the system's memory problems quickly and accurately?

There are certainly ways to do this. It's the same keyword, finding correlations.
In fact, although there are many memory performance metrics, they are all designed
to describe the principles of memory, and naturally the metrics are not completely
isolated from each other, but are generally correlated. This is why I always
emphasize the importance of the underlying principles and intersperse them in my
articles.

To take the simplest example, when you see that the remaining memory on the system
is low, does it mean that the process must not request to allocate new memory?
Of course not, because the memory available to the process includes recyclable
caches and buffers in addition to the remaining memory.

So, **to quickly pinpoint memory problems, I usually start by running a few performance
tools that cover a large area, such as `free`, `top`, `vmstat`, `pidstat`, etc**.

The specific analysis ideas are mainly these steps.

1. First, use `free` and `top` to see the overall memory usage of the system.
2. Then use `vmstat` and `pidstat` to see the trend over time to determine the type of memory problem.
3. Finally, detailed analysis is performed, such as memory allocation analysis, cache/buffer analysis, process-specific memory usage analysis, etc.

Also, I have drawn this analysis process into a flow chart that you can save and
print out to use.

<center><a href="https://ibb.co/h22Jk3b">
    <img src="https://i.ibb.co/MPPxbFr/Flowchart-Of-Memory-Bottleneck-Analysis.png" alt="Flowchart-Of-Memory-Bottleneck-Analysis" border="0">
</a></center>

The figure lists a few of the most commonly used memory tools, and the associated
analysis process. Where the arrows indicate the direction of analysis, a few examples
may be easier for you to understand.

For the first example, after you have gone through `free` and found that most of
the memory is occupied by cache, you can use `vmstat` or `sar` to observe the
trend of cache changes and confirm whether the cache usage continues to increase.

If it continues to grow, then the process causing the cache to rise is still running,
and then you can use a cache/buffer analysis tool (such as `cachetop`, `slabtop`, etc.)
to analyze exactly where those caches are being used.

In the second example, when you use the `free` command to check the memory of the
system and find out that there is not enough memory available, you should first
check if the memory is occupied by cache/buffer. After eliminating the cache/buffer,
you can go ahead and use `pidstat` or `top` to locate the process that is using
the most memory.

Once you find the process, you can then analyze the memory usage in the process
address space by using a process memory space tool (such as `pmap`).

In the third example, when you find that the memory is growing through `vmstat`
or `sar`, you can analyze whether there is a memory leak.

For example, you can use the memory allocation analysis tool `memleak` to check
if there is a memory leak. If there is a memory leak, `memleak` will output the
process and call stack of the memory leak for you.

Note that I have not listed all the performance tools in this diagram, but only
given the most core ones. In doing so, on the one hand, I really don't want the
extensive list of tools to scare you.

On the other hand, I hope you can focus on the core tools first and master the
usage and analysis ideas through the cases I provide and the practice in real
environments. After all, by mastering them proficiently, you will be able to
solve most memory problems.

## Summary

In today's article, I take you through a review of common memory performance metrics,
sort through common memory performance analysis tools, and finally summarize ideas
for quickly analyzing memory problems.

Although there are quite a lot of memory performance metrics and performance tools,
after understanding the fundamentals of memory management, you will find that
they are actually all related in some way. It is not difficult to sort out their
relationships and master the set of memory analysis.

After finding the source of the memory problem, the next step is to optimize it
accordingly. In my opinion, the most important thing about memory tuning is to
make sure that the application's hot data is put into memory and to minimize page
swapping and swaps.

There are so many common optimization ideas.

1. If you must enable Swap, lower the value of swappiness to reduce the tendency to use Swap during memory reclamation.
2. Reduce the dynamic allocation of memory. For example, you can use memory pools, large pages (HugePage), etc.
3. Try to use caches and buffers to access data. For example, you can use a stack to explicitly declare memory space to store data that needs to be cached, or use an external caching component such as Redis to optimize access to data.
4. Use cgroups, for example, to limit the memory usage of processes. This ensures that system memory is not exhausted by abnormal processes.
5. Tune the oom_score of the core app via <ins>/proc/pid/oom_adj</ins>. This ensures that the core app is not killed by OOM even when memory is tight.

