# 11 Routines: How to quickly analyze the bottleneck of the system CPU?

In the previous sections, I took you through several cases to analyze various
common CPU performance problems. Through these, I believe that you are no longer
unfamiliar and afraid of CPU performance analysis. At least you have a basic idea
and know a lot of CPU performance analysis tools.

However, I guess you may have also encountered a confusion I had: There are so
many CPU performance indicators, and there are a lot of CPU performance analysis
tools. If I leave the course and replace it with an actual work scene, I should
observe it again. What indicators, which performance tool to choose?

Don't worry, today I will summarize for you a "fast and accurate" bottleneck
positioning routine based on years of performance optimization experience, and
tell you how to choose indicator tools and how to find performance bottlenecks
in different scenarios.

## CPU performance indicators

Let's first review what are the performance indicators that describe the CPU.
You can find a piece of paper and write it from memory; or open the previous
article and summarize it yourself.

First of all, **the easiest thing to think of is CPU usage**, which is also the
most common performance indicator in the actual environment.

CPU usage describes the percentage of non-idle time in the total CPU time.
According to the different tasks running on the CPU, it is divided into user CPU,
system CPU, waiting for I/O CPU, soft interrupt and hard interrupt, etc.

- User CPU usage, including user-mode CPU usage (user) and low-priority user-mode CPU usage (nice), indicating the percentage of time the CPU is running in user mode. High user CPU usage usually indicates a busy application.
- System CPU usage, indicating the percentage of time the CPU is running in kernel mode (excluding interrupts). High system CPU usage indicates that the core is busy.
- CPU usage waiting for I/O, also commonly referred to as iowait, indicates the percentage of time waiting for I/O. If iowait is high, it usually means that the I/O interaction time between the system and the hardware device is relatively long.
- The CPU usage of soft interrupts and hard interrupts indicates the percentage of time that the kernel calls soft interrupt handlers and hard interrupt handlers, respectively. Their high usage usually indicates a high number of outages on the system.
- In addition to the above, there are also steal CPU usage (steal) and guest CPU usage (guest) that will be used in a virtualization environment, which respectively indicate the percentage of CPU time occupied by other virtual machines and the CPU running the guest virtual machine percentage of time.

**The second one that is easier to think of should be Load Average**, which is
the average number of active processes in the system. It reflects the overall
load of the system and mainly includes three values, which respectively refer to
the average load of the past 1 minute, the past 5 minutes and the past 15 minutes.

Ideally, the load average is equal to the number of logical CPUs, which means
that each CPU is exactly fully utilized. If the average load is greater than the
number of logical CPUs, it means that the load is heavy.

**The third one, which you may not notice before the course study, is process
context switching**, including:

- Voluntary context switches due to inability to acquire resources;
- Involuntary context switches caused by system-forced scheduling.

Context switching itself is a core function to ensure the normal operation of Linux.
However, excessive context switching will consume the CPU time of the original
running process on saving and restoring data such as registers, kernel stacks,
and virtual memory, shortening the real running time of the process and becoming
a performance bottleneck.

In addition to the above, there is another indicator, the CPU cache hit rate.
Since the development speed of CPU is much faster than that of memory, the
processing speed of CPU is much faster than the access speed of memory. In this
way, when the CPU accesses the memory, it is inevitable to wait for the response
of the memory. In order to coordinate the huge performance gap between the two,
CPU cache (usually multi-level cache) appeared.

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/R4Gd76m/CpuCache.png" alt="CpuCache" border="0">
</a></center>

As shown in the above picture, the speed of the CPU cache is between the CPU and
the memory, and the cache is the hot memory data. According to the ever-growing
hotspot data, these caches are divided into L1, L2, L3 these three-level caches
according to their sizes. Among them, L1 and L2 are commonly used in single-core,
and L3 is used in multi-core.

From L1 to L3, the size of these three level cache increases sequentially, and
correspondingly, the performance decreases sequentially (of course it is much
better than memory). And their hit rate measures the multiplexing of CPU cache.
The higher the hit rate, the better the performance.

These indicators are very useful and you need to be mastered them, so I summarized
them into a picture to help you classify and remember them. You can save and
print it, review it at any time, or use it as an "indicator screening" list for
CPU performance analysis.

<center><a href="https://ibb.co/YWVTmvy">
    <img src="https://i.ibb.co/th1Pgyx/Cpu-Indicators.png" alt="Cpu-Indicators" border="0">
</a></center>

## Profiling tools

After mastering the performance indicators of the CPU, we also need to know how
to obtain these indicators, that is, the use of tools.

Do you remember which tools were used in the previous cases? Here we also review
CPU performance tools together.

First, the load average case. We first used `uptime` to check the average load
of the system; after the average load increased, we used `mpstat` and `pidstat`
to observe the CPU usage of each CPU and each process respectively, and then
found out the cause of the average load increase. process, which is our pressure
testing tool `stress`.

Second, the case of context switching. We first used `vmstat` to check the
number of context switches and interrupts in the system; then through `pidstat`,
we observed the voluntary and involuntary context switches of the process; finally,
through `pidstat`, we observed the context switches of threads and found out the
context The source of the increased switching times is our benchmarking tool `sysbench`.

Third, the case of increased CPU usage by a process. We first use `top` to check
the CPU usage of the system and processes, and find that the process with increased
CPU usage is php-fpm; then use `perf top` to observe the call chain of php-fpm,
and finally find out the root cause of the increased CPU. That is, the library
function sqrt().

Fourth, the case where the CPU usage of the system increases. We first used `top`
to observe the increase of the system CPU, but through `top` and `pidstat`, we
could not find the process with high CPU usage. Therefore, we re-examined the
output of `top`, and started with the process with low CPU usage but in the
Running state, and found out suspicious things. Finally, through `perf record`
and `perf report`, we found that it were short-term processes that were causing trouble.

In addition, for short-term processes, I also introduced a special tool `execsnoop`,
which can monitor the external commands called by the process in real time.

Fifth, the case of uninterruptible processes and zombie processes. We first used
top to observe the problem of iowait rising, and found a large number of uninterruptible
processes and zombie processes; then we used `dstat` to find that this was caused
by disk reading, so we found out related processes through `pidstat`. However,
we failed to use `strace` to view the process system calls. Finally, we used
`perf` to analyze the process call chain, only to find that the root cause was
disk direct I/O.

The last one, the case of soft interrupts. We observed through `top` that the
CPU usage of softirqs in the system increased; then we checked <ins>/proc/softirqs</ins>
and found several softirqs with a faster rate of change; then through the `sar`
command, we found that it was a problem with small network packets, and finally
used `tcpdump`, find out the type and source of the network frame, and determine
that it is caused by a SYN FLOOD attack.

At this point, I guess you are already dizzy. It turns out that in just a few cases,
we have used more than a dozen CPU performance tools, and the applicable scenarios
of each tool are different! How to distinguish between so many tools? In the
actual performance analysis, how to choose?

My experience is to understand them from two different dimensions, so that they
can be used flexibly.

### Learn and use flexibly, linking performance indicators and profiling tools

**The first dimension starts from the performance indicators of the CPU. In other
words, when you want to look at a performance indicator, you need to know which
tools can do it.**

According to different performance indicators, classify and understand the
performance tools that provide indicators. In this way, when actually troubleshooting
performance problems, you can clearly know which tools can provide the indicators
you want, instead of trying one by one without any basis and hitting luck.

In fact, I have used this idea many times in previous cases. For example, after
using `top` to find out that the CPU usage of soft interrupts is high, the next
step is naturally to know the specific soft interrupt types. Where can I observe
the operation of various soft interrupts? Of course it is the <ins>/proc/softirqs</ins>
file in the proc file system.

Then, for example, the type of soft interrupt we found is network reception,
so we must continue to think in the direction of network reception. What is the
system's network reception like? What tools can check network reception? In our
case, it was `dstat` that was used.

Although you don't need to memorize all the tools, if you can understand the
characteristics of the tools corresponding to each indicator, you will be able
to use them more efficiently and flexibly. Here, I have made a table of the tools
that provide CPU performance indicators, which is convenient for you to sort out
the relationship and understand the memory. Of course, you can also use it as a
"metric tool" guide.

<center><a href="https://ibb.co/VMgNRcL">
    <img src="https://i.ibb.co/nQ709Nw/Find-Tool-By-Indicator-Cpu.png" alt="Find-Tool-By-Indicator-Cpu" border="0">
</a></center>

Next, let's look at the second dimension.

**The second dimension starts from tools. That is, when you have installed a
certain tool, you need to know what indicators this tool can provide.**

This is also very important in the actual environment, especially in the production
environment, because in many cases, you do not have permission to install new
toolkits, and you can only maximize the use of the tools already installed in
the system, which requires you to know them enough.

Specific to the usage of each tool, it generally supports rich options. But don't
worry, these configuration options don't need to be memorized. You just need to
know what tools are available and what the basic functions of these tools are.
When you really want to use them, you can use the `man` command to check their manuals.

Similarly, I have also compiled these commonly used tools into a table for you to
distinguish and understand. Naturally, you can also use it as a "tool indicator"
guide, just look up the table when needed.

<center><a href="https://ibb.co/cYMHK14">
    <img src="https://i.ibb.co/1zh5YLF/Find-Indicator-By-Tool-Cpu.png" alt="Find-Indicator-By-Tool-Cpu" border="0">
</a></center>

## How to quickly analyze CPU performance bottlenecks

I believe that at this point, you are already very familiar with the performance
indicators of the CPU, and you know which tools can be used to obtain each performance
indicator.

Does that mean that every time you encounter a CPU performance problem, you have
to run all the above tools, and then analyze all the CPU performance indicators?

You probably feel that this simple way of searching is like searching stupidly.
Don't joke though, because that's what I did in the first days. It is of course
possible to find out all the indicators and analyze them in a unified manner,
and it is also possible to find potential bottlenecks in the system.

But the efficiency of this method is really too low! Not to mention time-consuming
and labor-intensive, in the face of a huge indicator system, you may accidentally
overlook a certain detail, resulting in a waste of work. I have suffered so many times.

Therefore, in the actual production environment, we usually hope to locate the
bottleneck of the system as quickly as possible, and then optimize performance
as quickly as possible, that is, to solve performance problems quickly and accurately.

Is there any way to quickly and accurately find the bottleneck of the system?
The answer is yes.

Although there are many CPU performance indicators, you must know that since
they all describe the CPU performance of the system, they will not be completely
isolated, and many indicators are related to each other. **If you want to understand
the correlation of performance indicators, you must understand how each performance
indicator works**. This is why when I introduce each performance indicator, I have
to explain the relevant system principles interspersed. I hope you can remember this.

For example, if the user CPU usage is high, we should check the user space of the
process instead of the kernel space. Because the user CPU usage reflects the CPU
usage in the user space, and the CPU usage in the kernel space will only be
reflected in the system CPU usage.

You see, with this basic understanding, we can narrow down the scope of our
investigation, saving time and effort.

Therefore, in order to **narrow the scope of investigation, I usually run several
tools that support more indicators, such as `top`, `vmstat` and `pidstat`**. Why
these three tools? Take a closer look at the picture below and you will understand.

<center><a href="https://ibb.co/qFXnPnG">
    <img src="https://i.ibb.co/VprNPNy/First-Step-Tools-Cpu.png" alt="First-Step-Tools-Cpu" border="0">
</a></center>

In this picture, I list the important CPU indicators provided by `top`, `vmstat`
and `pidstat` respectively, and use dotted lines to indicate the correlation,
which corresponds to the next step of performance analysis.

From this picture, you can find that these three commands contain almost all
important CPU performance indicators, such as:

- You can get various CPU usage and zombie processes and load average information from the output of `top`.
- From the output of `vmstat` you can get the number of context switches, interrupts, running and uninterruptible processes.
- From the output of `pidstat`, you can get the user CPU usage, system CPU usage, and voluntary and involuntary context switches of the process.

In addition, many indicators output by these three tools are related to each other,
so I also use a dotted line to indicate their relationship. It may be easier for
you to understand with a few examples.

In the first example, the process user CPU usage output by `pidstat` increases,
which will cause the user CPU usage output by `top` to increase. Therefore, when
you find that there is a problem with the user CPU usage output by `top`, you can
compare it with the output of `pidstat` to see whether the problem is caused by
a certain process.

After finding out the process that caused the performance problem, it is necessary
to use process analysis tools to analyze the behavior of the process, such as
using `strace` to analyze the system call situation, and using `perf` to analyze
the execution of functions at all levels in the call chain.

In the second example, the increase in the average load output by `top` can be
compared with the number of running state processes and the number of uninterruptible
state processes from output by `vmstat` to observe which process causes the load increase.

- If the number of uninterruptible processes increases, then I/O analysis is required, that is, tools such as `dstat` or `sar` are used to further analyze the I/O situation.
- If the number of running processes has increased, you need to go back to `top` and `pidstat` to find out what processes are running, and then use the process analysis tool for further analysis.

As the last example, when you find that the CPU usage of the softirqs output by
`top` has increased, you can check the changes of various types of softirqs in
the <ins>/proc/softirqs</ins> file to determine which softirq is causing the problem.
For example, if it is found that the problem is caused by the interruption of
network reception, then you can continue to use the network analysis tools `sar`
and `tcpdump` to analyze.

Note that I only list the core profiling tools in this figure, not all of them.
This is done partly because you don't want to overwhelm you with a huge list of
tools. It is not necessarily a good thing to get in touch with all or core or
niche tools at the beginning of learning. On the other hand, I hope you can focus
on the core tools first. After all, you can solve most problems by mastering them.

Therefore, you can save this picture as a mind map for CPU performance analysis.
Starting from the core tools, through the cases I provided, practice in the real
environment and win them.

## Summary

Today, I will take you to recall common CPU performance indicators, sort out
common CPU performance observation tools, and finally summarize the ideas for
quickly analyzing CPU performance problems.

Although there are many CPU performance indicators and corresponding performance
analysis tools, after you are familiar with the meaning of various indicators,
you will find that they are actually related to a certain extent. Following this
line of thought, it is not difficult to master commonly used analysis routines.