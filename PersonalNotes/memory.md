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