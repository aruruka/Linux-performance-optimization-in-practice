## How to count the physical memory usage of all processes?

Of course, there are students who have very correct ideas, such as JohnT3e mentioned, the key point of this question is to understand the meaning of PSS.

> The key point of the final reflection question is to understand the meaning of the PSS terms. Interested students can refer to this source:
> https://unix.stackexchange.com/questions/33381/getting-information-about-a-process-memory-usage-from-proc-pid-smaps

You can of course find the answer via the link on stackexchange, but I would recommend, moreover, looking directly at the 🔗[documentation](https://www.kernel.org/doc/Documentation/filesystems/proc.txt) for the proc file system:

> The “proportional set size” (PSS) of a process is the count of pages it has in memory, where each page is divided by the number of processes sharing it.
> So if a process has 1000 pages all to itself, and 1000 shared with one other process, its PSS will be 1500.

Let me briefly explain here that the PSS of each process is the sum of the shared memory divided equally among the processes, plus the non-shared memory size of the process itself.

Like this example in the documentation, a process has 1000 pages of non-shared memory, and it shares 1000 pages with another process, so its PSS = 1000/2 + 1000 = 1500 pages.

This way, you can just accumulate the PSS and not worry about double counting the shared memory.

For example, you can run the following command to calculate:

```shell
# 使用grep查找Pss指标后，再用awk计算累加值
$ grep Pss /proc/[1-9]*/smaps | awk '{total+=$2}; END {printf "%d kB\n", total }'
391266 kB
```

## Is there was a way to see the buffer/cache of each process?

I'm recently learning Linux performance analysis. There is a case study regarding buffer and cache in memory, and it got me thinking about the question of how Hadoop or MapR FS accesses disks and maintains their buffer/cache as a file system themselves.
So I wondered if there was a way to see the buffer/cache of each process from the OS level, but according to 🔗[this StackOverflow post](https://lnkd.in/g7bMtRUr), the buffer/cache doesn't belong to the process but to the kernel.
I later rediscovered these sentences in 🔗[this article](https://lnkd.in/gUET59Mc) I had studied:

> Avoid double counting the memory occupied by multiple processes simultaneously, such as page cache, shared memory, and so on.
> If you add up the data from ps and top directly, you will have a double-counting problem.

For example, if you add up the RES of the processes output by `top`, it will normally be greater than the total DRAM.
So, this also confirms that the above StackOverflow post is correct, i.e. the buffer/cache is shared by multiple processes.

So, for Hadoop, it seems that people always need to pre-format the disks as some kind of file system (ext3, xfs, etc.), hence Hadoop shouldn't use direct I/O to manipulate the disks(but maybe using RAW I/O).
So, it is more likely that the Hadoop file system calls the OS system calls without direct I/O to access the disks and therefore creates a buffer/cache.

On the other hand, MapR FS handles raw devices so I think it uses direct I/O to manipulate the disks. Naturally, it doesn't necessarily use system calls with cached I/O to read from and write to the disks, just like most database systems.
(As far as I know, Oracle uses direct I/O to manipulate disks.)
So MapR FS should be maintaining its own buffer/cache system inside itself, and the buffer/cache data may be stored inside the heap within its virtual memory.

Besides, for a distributed file system, it is quite possible to maintain an efficient caching system (similar to Redis) within itself.

BTW, direct I/O is skipping buffer/cache, raw I/O is skipping the file system (which still has a buffer).