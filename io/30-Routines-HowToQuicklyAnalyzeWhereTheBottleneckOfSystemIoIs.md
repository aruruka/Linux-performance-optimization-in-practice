# 30 Routines: How To Quickly Analyze Where The Bottleneck Of System I/O Is?

In the previous sections, we have analyzed various common I/O performance problems through several case studies. Through these real-world operations, you should have become familiar with the analysis and positioning ideas of I/O performance problems, and also mastered many tools for I/O performance analysis.

However, I think you may still be confused, if you leave the column and switch to other practical work scenarios, how to choose the various performance metrics and tools mentioned in the case?

At the end of the last section, I left the homework for you to organize your own thoughts. Today, I will take you to review and summarize how to locate the I/O bottleneck of the system quickly and accurately, and sort out how to choose the metrics and tools and how to locate the performance bottleneck in different scenarios.

## Profiling metrics

As usual, let's review what are the performance metrics that describe I/O. You can start by thinking back to the principles of file system and disk I/O and write your own from memory and understanding with this Linux system I/O stack diagram below. Or, you can open the previous articles and review them one by one to summarize.

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/3fGP12X/Architecture-Diagram-Of-Linux-File-System.png" alt="Architecture-Diagram-Of-Linux-File-System" border="0">
</a></center>

After learning about I/O performance for so long, when you talk about I/O metrics, you should first think of categorical descriptions. We need to distinguish between file systems and disks and use separate metrics to describe their performance.

## File system I/O profiling metrics

Let's look at the file system first.

**First, the easiest thing to think of is storage space usage, including capacity, usage, and space remaining.**
We usually also call these disk space usage, because the data of the file system is ultimately stored on disk.

Note, however, that these are only the space usage that the file system shows to the outside world, not the real usage in disk space, because the file system's metadata also takes up disk space.

Also, if you configure RAID, the amount of usage you see from the file system and the amount of space actually taken up on the disk will be different depending on the RAID level. For example, with RAID 10 configured, you will only see up to half of the capacity of all disks from the file system.

In addition to the storage space of the data itself, there is another **easy to ignore is the usage of index nodes, which also includes three indicators such as capacity, usage, and remaining amount**. If too many small files are stored in the file system, you may run into the problem that the index node capacity is full.

**Secondly, you should think of the cache usage mentioned many times before, including page cache, directory item cache, index node cache, and the cache of each specific file system (such as ext4, XFS, etc.)**. These caches will use faster memory for temporary storage of file data or file system metadata, which can reduce the number of accesses to slow disks.

In addition to these two points, file I/O is also an important performance metric, including IOPS (both r/s and w/s), response time (latency), and throughput (B/s). When examining such metrics, it is common to also consider the actual file reads and writes. For example, file I/O performance is analyzed in the context of file size, number of files, I/O types, and so on.

It is true that these performance metrics are very important, but unfortunately, the Linux file system does not provide a way to look at these metrics directly. We can only observe and evaluate them indirectly through system calls, dynamic tracing or benchmarking.

However, in practice, these metrics are easier to see when we look at disk performance because Linux provides more detailed data for disk performance.

## Disk I/O profiling metrics

Next, let's look specifically at what performance metrics measure the performance of disk I/O.

In the article on disk I/O principles, I mentioned four core disk I/O metrics.

- **Utilization**, which is the percentage of disk that is busy processing I/O requests. Excessive utilization (e.g., over 60%) usually means there is a performance bottleneck in disk I/O.
- **IOPS** (Input/Output Per Second), which is the number of I/O requests per second.
- **Throughput**, which is the size of I/O requests per second.
- **Response Time**, which is the interval from the time an I/O request is sent to the time a response is received.

When examining these metrics, be sure to pay attention to the specific scenarios of integrated I/O to analyze, such as read and write types (sequential or random), read and write ratios, read and write sizes, storage types (with or without RAID and RAID levels, local storage or network storage), and so on.

However, there is a big taboo here, which is to analyze and compare I/O performance metrics of different scenarios directly. This is a very common misconception that you should definitely avoid.

In addition to these metrics, as I mentioned several times in the previous articles on Cache and Buffer principles, **Buffer** is also a key metric to master, and it often appears in the analysis of memory and disk problems.

These metrics for both file system and disk I/O are useful and need to be mastered, so I've summarized them into a chart to help you sort and remember them. You can save and print it out for easy review at any time, or use it as a "metrics filter" list for I/O performance analysis.

<center><a href="https://ibb.co/4pG9qMq">
    <img src="https://i.ibb.co/qpfXS9S/IO-Profiling-Metrics.png" alt="IO-Profiling-Metrics" border="0">
</a></center>

## Profiling tools

After mastering the performance metrics of the file system and disk I/O, we also need to know how to get these metrics, that is, to understand the use of tools.

Do you remember which tools were used in the previous basic and case studies? Let's review these contents together.

First, in the principle of the file system, I introduced the tool to view the file system capacity `df`. It can view both the spatial capacity of the file system data and the capacity of the index nodes. As for the file system cache, we look at the page cache, directory item cache, index node cache, and specific file system cache through various sources such as <ins>/proc/meminfo</ins>, <ins>/proc/slabinfo</ins>, and `slabtop`.

Second, in the principle of disk I/O, we looked at disk and process I/O with `iostat` and `pidstat`, respectively. They are both the most commonly used tools for I/O performance analysis. With `iostat`, we can get performance metrics such as disk I/O usage, throughput, response time, and IOPS, while with `pidstat`, we can observe process I/O throughput and block device I/O latency, etc.

Third, in the case of wild logging, we first used `top` to check the CPU usage of the system and found that iowait was relatively high; then, we used `iostat` to find the bottleneck of disk I/O usage, and used `pidstat` to find the process with a lot of I/O; finally, by using `strace` and `lsof`, we found the file that the problem process was Finally, using `strace` and `lsof`, we identified the files that the problem process was reading and writing, and finally pinpointed the source of the performance problem - it turned out to be a process that was logging furiously.

Fourth, in the word hotness case of disk I/O latency, we also first used `top` and `iostat` to find out that there was an I/O bottleneck on the disk, and used `pidstat` to find out the process with a lot of I/O. But then, trying to replicate the last operation, we failed. In the subsequent `strace` command, we didn't see the write system call. So, we changed our thinking and used the new tools `filetop` and `opensnoop` to trace the system calls from the kernel and eventually find the source of the bottleneck.

Finally, in the case of MySQL and Redis, we used `top`, `iostat`, and `pidstat` to identify and locate the bottlenecks of the I/O performance problem, which were mysqld and redis-server, and then we used `strace`+`lsof` to find the files they were reading and writing. files they were reading and writing.

In the MySQL case, based on the paths of the files being read and written by mysqld and the principles of the MySQL database engine, we not only found the names of the databases and data tables, but also further discovered the slow query problem, and finally solved the performance bottleneck by optimizing the indexes.

As for the Redis case, based on the files read and written by the redis-server and the TCP sockets that were communicating over the network, combined with the working principle of Redis, we found that there was a problem with the Redis persistence option configuration; from the TCP socket communication data, we also found unreasonable behavior of the client. So, we modified the Redis configuration options and optimized the way the client uses Redis to reduce the number of network communications and solve the performance problem.

After reviewing all this, you feel dizzy and want to sigh again at the complexity of performance tools! In fact, as long as you understand how the corresponding system works, it is not difficult to use the tools

## Linkage of performance indicators and tools

As with the previous CPU and memory sections, I recommend organizing your memory in two different dimensions: metrics and tools.

- Starting with I/O metrics makes it easier for you to relate performance tools to how the system works and to get a macro view and grasp of performance issues.
- Starting with performance tools, on the other hand, allows you to get started with the tools more quickly and find out the performance metrics we want to observe quickly. Especially in the case of limited tools, we have to make full use of every tool at hand, and a small number of tools should try to dig out a lot of information.

**The first dimension, starting with file system and disk I/O performance metrics. In other words, when you want to look at a certain performance metric, know exactly, which tools can do it.**

Categorize and understand the performance tools that provide metrics based on different performance metrics. That way, when actually troubleshooting performance issues, you know exactly what tools can provide the metrics you want, rather than trying one by one without any basis and bumping your luck.

Although you don't need to memorize all the relevant tools, if you can remember the characteristics of the tools corresponding to each metric, you can be more efficient and flexible in practice.

Here, I have made a table of tools that provide I/O performance metrics, so that you can easily sort out the relationships and understand the memory. You can save it and print it out to remember at any time. Of course, you can also use it as a "metrics tool" guide.

<center><a href="https://ibb.co/ZY8T0H9">
    <img src="https://i.ibb.co/Gt2Pqsm/Find-Tools-Based-On-Indicators-File-System-And-Disk-Io.png" alt="Find-Tools-Based-On-Indicators-File-System-And-Disk-Io" border="0">
</a></center>

Next, let's look at the second dimension.

**The second dimension, starting with the tool. That is, when you already have a tool installed, know what metrics the tool provides.**

This is also very important in real-world environments, especially in production environments. Because in many cases, you don't have access to install new toolkits and can only maximize the use of the tools already on your system, and this requires you to know them well enough.

Specific to the use of each tool, there are generally rich configuration options supported. But don't worry, these configuration options don't have to be memorized. You just need to know what tools are available and what the basic functions of these tools are. When you really need to use them, just check their manuals via the `man` command.

Again, I've put together a table of these common tools to help you distinguish and understand them. Naturally, you can also use it as a "tool indicator" guide, and just look up the table when you need it.

<center><a href="https://ibb.co/X3BQfkw">
    <img src="https://i.ibb.co/hZ56x1q/Check-Metrics-Against-Tools-File-System-And-Disk-Io.jpg" alt="Check-Metrics-Against-Tools-File-System-And-Disk-Io" border="0">
</a></center>

## How to quickly analyze I/O performance bottlenecks

By now, I believe you are very familiar with the memory performance metrics and know what tools can be used to get each performance metric.

You should have noticed that, compared to the first two panels, although there are still more I/O performance metrics for file systems and disks, there are actually only a few core performance tools. Master them, and then according to the actual system phenomenon, and with the system and application principles, I/O performance analysis is very clear.

However, no matter what, if you run through all the tools mentioned above every time you encounter I/O performance problems, it is definitely not realistic.

In a real production environment, what we want is to locate the bottleneck of the system as **fast** as possible, and then optimize the performance as **fast** as possible, that is, to solve the performance problem quickly and accurately.

So is there any way to find out the I/O bottleneck of the system quickly and accurately? The answer is yes.

As always, look for correlations. There is some correlation between multiple performance metrics, so don't look at them in complete isolation. **To understand the correlation of performance metrics, you need to understand how each performance metric works**. This is why I intersperse each performance metric with an explanation of the relevant system principles, and again, I hope you'll keep this in mind.

Take our previous installments as an example. If you carefully compare the previous cases, from the I/O latency case to the MySQL and Redis cases, you will see that although these problems are very different, the analysis from the I/O perspective starts with basically similar analysis ideas, which are:

1. first use iostat to find disk I/O performance bottlenecks;
2. then use pidstat to locate the processes causing the bottlenecks;
3. then analyze the I/O behavior of the processes;
4. finally, analyze the source of these I/Os in the context of the application's rationale.

**So, to narrow down the troubleshooting, I usually run those few tools that support more indicators first, such as `iostat`, `vmstat`, `pidstat`, etc.** Then, based on the observed phenomena, I look for the next step of analysis in conjunction with the system and application principles. I have drawn this process into a diagram, which you can save and use for reference.

<center><a href="https://ibb.co/qjsDRfw">
    <img src="https://i.ibb.co/MSVgR4L/Flowchart-IO-Performance-Check.png" alt="Flowchart-IO-Performance-Check" border="0">
</a></center>

The figure lists the most commonly used file system and disk I/O performance analysis tools and the corresponding analysis process, with the arrows indicating the direction of analysis. Among them, `iostat`, `vmstat`, and `pidstat` are the core performance tools, which also provide the most important I/O performance metrics. A few examples may be easier to understand.

For example, in the MySQL and Redis cases we talked about earlier, we used `iostat` to identify I/O performance bottlenecks on disk, then used `pidstat` to find the process with the largest I/O, then used `strace` to find the files being read and written by that process, and finally combined with the application principles to find the cause of the large amount of I/O.

For example, when you use `iostat` to find an I/O performance bottleneck on disk, then check with `pidstat` and `vmstat`, you may find that the I/O is coming from a kernel thread, such as Swap using a lot of elevation. In this case, you'll have to do a memory analysis to find the processes that are using a lot of memory and then try to reduce the memory usage.

Also note that I have listed only the core performance tools in this chart, not all the tools in the previous table. This is done, on the one hand, because I don't want to scare you with a large list of tools. It's not necessarily good to be exposed to all the core or niche tools at the beginning of your learning. On the other hand, you are also expected to focus on the core tools first, after all, being proficient with them will solve most problems.

So, you can save this chart as a mapping of ideas for file system and disk I/O performance analysis. Start with these core tools and practice them yourself in a real environment with those cases I provided to take them down.

## Summary

Today, we reviewed common file system and disk I/O performance metrics together, sorted through common I/O performance observation tools, and established the correlation between performance metrics and tools. Finally, we also summarized ideas for quickly analyzing I/O performance problems.

As usual, although there are many I/O performance metrics and corresponding performance analysis tools, you will naturally find their correlation once you are familiar with the meaning of each metric. Going down the road, it is not difficult to master the common analysis routines.
