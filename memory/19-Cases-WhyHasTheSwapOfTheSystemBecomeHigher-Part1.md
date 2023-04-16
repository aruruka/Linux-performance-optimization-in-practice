# 19 Cases： Why has the Swap of the system become higher (Part 1)

In the last section, I took you through a Fibonacci sequence case study to learn
about memory leak analysis. If dynamic memory is allocated directly or indirectly
in your program, you must remember to release them, otherwise it will lead to memory
leaks and even exhaust the system memory in severe cases.

However, on the flip side, what does the system do when a memory leak occurs, or
when a large memory application is run, causing the system's memory resources to
be strained?

As we have learned in the memory fundamentals chapter, this actually leads to two
possible outcomes, memory reclamation and OOM killing the process.

Let's look at the latter possible outcome, OOM (Out Of Memory) due to memory
resource constraints, which is relatively easy to understand and refers to the
system killing processes that are taking up a lot of memory, releasing that memory,
and reallocating it to other processes that need it more.

Let's look at the latter possible outcome, OOM (Out Of Memory) due to memory
resource constraints, which is relatively easy to understand and refers to the
system killing processes that are taking up a lot of memory, releasing that memory,
and reallocating it to other processes that need it more.

We have talked about this in detail earlier, so we won't repeat it here.

Next, let's look at the first possible outcome, memory reclamation, which means
that the system releases memory that can be reclaimed, such as the caches and
buffers I talked about earlier, which are reclaimable memory. They are often
called **File-backed Pages** in memory management.

Most of the file pages can be recycled directly and just re-read from disk later
when needed. Those that have been modified by the application and have not been
written to disk for a while (i.e., dirty pages) will have to be written to disk
before memory can be freed.

These dirty pages, in general, can be written to disk in two ways.

- Dirty pages can be synchronized to disk in the application by means of a systemcall to `fsync`;
- Can also be given to the system, by the kernel thread `pdflush` responsible for these dirty pages refresh.

In addition to caches and buffers, a file-mapped page, obtained through memory
mapping, is also a common file page. It can also be freed and re-read from the
file the next time it is accessed.

Is there any other memory that can be reclaimed besides file pages? For example,
is it possible to reclaim heap memory that is dynamically allocated by the application,
which we talk about in memory management as **Anonymous Page**?

I'm sure you're saying that they're likely to be accessed again, so of course they
can't be recycled directly. That's very true. Naturally, this memory cannot be
freed directly.

However, it seems to be a waste of resources if this memory is rarely accessed
after it is allocated. Isn't it possible to store them temporarily on disk first
and free up memory for other processes that need it more?

Swap writes this infrequently accessed memory to disk and then frees it up for
other processes that need it more. When you access the memory again, you just
read it from disk again.

In the previous sections, we have learned the principles and analysis of caching
and OOM respectively. How does Swap work? Because there is more content, next,
I will use two lessons to take you to explore how Swap works and how Swap is
analyzed after it is elevated.

Today we're going to take a look at how Swap actually works.

## Swap Principle

As mentioned earlier, Swap is simply the process of taking a piece of disk space
or a local file (the following explanation uses disk as an example) and using it
as memory. It consists of two processes: swap out and swap in.

- By swapping out, we mean storing the memory data that the process is not using temporarily to disk and releasing the memory occupied by this data.
- The swap-in, on the other hand, reads them from disk to memory when the process accesses them again.

So you see, Swap actually makes the available memory on the system larger. This
makes it possible to run applications with large memory even if the server is running
low on memory.

I remember when I first learned Linux OS, memory was so expensive that an average
student couldn't afford to use large memory, so I turned on Swap to run the Linux
desktop at that time. Of course, nowadays memory is much cheaper and servers are
usually configured with a lot of memory, so does that mean Swap is no longer useful?

Of course not. In fact, even the largest amount of memory is not enough for an
application at times.

A typical scenario is that even when memory is running low, some applications do
not want to be killed by OOM, but want to take some time to wait for manual
intervention or for the system to automatically release memory from other processes
and allocate it again.

In addition, the hibernation and fast boot features of our common laptops are
also based on Swap. When hibernating, the system's memory is stored on the disk
so that when it is turned on again, the memory is simply loaded from the disk.
This eliminates the need to initialize many applications and speeds up the boot process.

That said, since Swap is about reclaiming memory, when exactly does Linux need to
reclaim memory? We've been talking about tight memory resources, but how do we
measure if memory is tight?

One of the easiest scenarios to think of is when there is a new request for a large
block of memory allocation, but there is not enough memory left. At this point
the system needs to reclaim a portion of memory (such as the cache mentioned earlier)
and thus satisfy the new memory request as much as possible. This process is often
referred to as **direct memory reclamation**.

In addition to direct memory reclamation, there is a dedicated kernel thread for
periodic memory reclamation, **kswapd0**. To measure memory usage, kswapd0 defines
three memory thresholds (watermarks, also known as water levels). They are pages_min,
pages_low, and pages_high. The remaining memory, in turn, is represented by pages_free.

Here, I have drawn a diagram to represent their relationship.

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/CMq8W7R/kswapd0-wartermark.png" alt="kswapd0-wartermark" border="0">
</a></center>

kswapd0 periodically scans for memory usage and performs memory reclamation operations
based on where the remaining memory falls in the space of these three thresholds.

- The remaining memory is less than the page minimum threshold, which means that the process is running out of available memory and only the kernel can allocate memory.
- If the remaining memory falls between the page minimum threshold and the page low threshold, it means that the memory pressure is high and there is not much memory left. At this point, kswapd0 performs memory reclamation until the remaining memory is greater than the high threshold.
- Remaining memory falls in the middle of the page low threshold and page high threshold, indicating that memory is under some pressure, but can still accommodate new memory requests.
- A remaining memory greater than the page high threshold indicates that there is more memory remaining and no memory pressure.

We can see that once the remaining memory is less than the page low threshold,
memory reclamation is triggered. This page low threshold can be set indirectly
through the kernel option <ins>/proc/sys/vm/min_free_kbytes</ins>. min_free_kbytes
sets the page minimum threshold, while the other two thresholds are calculated
based on the page minimum threshold and are calculated as follows:

```plaintext
pages_low = pages_min*5/4
pages_high = pages_min*3/2
```

## NUMA and Swap

In many cases, you may find that the Swap has increased, but when you analyze
the system's memory usage, you may find that there is still a lot of memory left.
Why does Swap occur even when there is a lot of memory left?

As you should have thought by the title above, this is the result of the processor's
NUMA (Non-Uniform Memory Access) architecture.

I mentioned briefly about NUMA in the CPU module. In the NUMA architecture,
multiple processors are partitioned onto different Nodes, and each Node has its
own local memory space.

The memory space inside the same Node can actually be further divided into
different memory zones (Zones), such as Direct Memory Access Zone (DMA), Normal
Memory Zone (NORMAL), Pseudo Memory Zone (MOVABLE), etc., as shown in the following
figure:

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/CQmdp32/NUMA-Zone.png" alt="NUMA-Zone" border="0">
</a></center>


For now, we don't need to pay special attention to the specific meaning of these
memory zones, it's enough that we will check the configuration of the thresholds
and the actual usage of the cache and anonymous pages.

Since each Node in the NUMA architecture has its own local memory space, when
analyzing memory usage, we should also analyze each Node individually.

You can use the `numactl` command to see how processors are distributed across
Nodes and how much memory each Node uses. For example, the following is an example
of `numactl` output:

```shell
$ numactl --hardware
available: 1 nodes (0)
node 0 cpus: 0 1
node 0 size: 7977 MB
node 0 free: 4416 MB
...
```

This screen shows that my system has only one Node, Node 0, and that the two CPUs,
numbered 0 and 1, are located on Node 0. In addition, Node 0 has 7977 MB of memory
and 4416 MB of memory remaining.

After understanding the architecture of NUNA and how NUMA memory is viewed, you
may ask what does this have to do with Swap?

In fact, the three memory thresholds mentioned earlier (pages_min threshold,
pages_low threshold, and pages_high threshold) can all be viewed through the
memory zone's interface <ins>/proc/zoneinfo</ins> in the proc file system.

For example, the following is an example of the contents of a <ins>/proc/zoneinfo</ins> file:

```shell
$ cat /proc/zoneinfo
...
Node 0, zone   Normal
 pages free     227894
       min      14896
       low      18620
       high     22344
...
     nr_free_pages 227894
     nr_zone_inactive_anon 11082
     nr_zone_active_anon 14024
     nr_zone_inactive_file 539024
     nr_zone_active_file 923986
...
```

There are a large number of metrics in this output, let me explain some of the
more important ones.

- min, low, and high at pages are the three memory thresholds mentioned above, while free is the number of free memory pages, which is the same as nr_free_pages later.
- nr_zone_active_anon and nr_zone_inactive_anon are the number of anonymous pages that are active and inactive, respectively.
- nr_zone_active_file and nr_zone_inactive_file, which are the number of active and inactive file pages, respectively.

From this output, we can see that the free memory is much larger than the pages_high
threshold, so kswapd0 will not reclaim memory at this point.

Of course, when a Node runs out of memory, the system can either find free memory
from other Nodes or reclaim memory from local memory. The exact mode you choose
can be adjusted via <ins>/proc/sys/vm/zone_reclaim_mode</ins>. It supports the
following options:

- The default of 0, which is the mode just mentioned, means that you can either find free memory from other Nodes or reclaim memory locally.
- 1, 2, and 4 all mean that only local memory is reclaimed, 2 means that memory can be reclaimed by writing back dirty data, and 4 means that memory can be reclaimed by swap.

## swappiness

By this point, we can understand the mechanism of memory reclamation. These
reclaimed memories include both file pages and anonymous pages. 

- Recycling of file pages means, of course, recycling the cache directly, or writing dirty pages back to disk and then recycling them.
- The recovery of anonymous pages is actually done through the Swap mechanism, which writes them to disk and then frees the memory.

However, you may still have a question. Since there are two different memory
recycling mechanisms, which one should be recycled first when it comes to actually
recycling memory?

In fact, Linux provides a <ins>/proc/sys/vm/swappiness</ins> option to adjust
the level of aggressiveness in using Swap.

The range of swappiness is 0-100, the higher the value, the more aggressive the
use of Swap, that is, the more inclined to recycle anonymous pages; the lower the
value, the more negative the use of Swap, that is, the more inclined to recycle
file pages.

Although the range of swappiness is 0-100, note though that this is not a percentage
of memory, but a weight to adjust how aggressive the Swap is, and even if you set
it to 0, the Swap will still occur 🔗[when the remaining memory + file pages is less
than the page height threshold](https://www.kernel.org/doc/Documentation/sysctl/vm.txt).

After getting clear about the Swap principle, how to locate and analyze when the
Swap usage becomes high? Don't worry, in the next section, we will use a case to
explore the practice.

## Summary

When memory resources are tight, Linux frees up file pages and anonymous pages
through direct memory reclamation and periodic scans so that memory can be
allocated to processes that need it more.

- The recycling of file pages is easier to understand, just empty them, or write the dirty data back to disk and then release them.
- The recovery of anonymous pages needs to be swapped out to disk via Swap, and then swapped in from disk to memory on the next access.

You can set <ins>/proc/sys/vm/min_free_kbytes</ins> to adjust the threshold at
which the system periodically reclaims memory (aka the page low threshold), and
you can set <ins>/proc/sys/vm/swappiness</ins> to adjust the tendency to reclaim
file pages and anonymous pages.

In the NUMA architecture, each Node has its own local memory space, and when local
memory runs low, by default it can either find free memory from other Nodes or
reclaim it from local memory.

You can set <ins>/proc/sys/vm/zone_reclaim_mode</ins> to adjust the NUMA local memory reclaim
policy.