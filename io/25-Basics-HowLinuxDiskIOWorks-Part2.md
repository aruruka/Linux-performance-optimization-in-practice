# 25 Basics: How Linux disk I/O works (part 2)?

In the previous section, we learned how Linux disk I/O works and learned about the Linux storage system I/O stack consisting of the file system layer, the common block layer, and the device layer.

The Common Block Layer is the core of Linux disk I/O. Upward, it provides a standard interface for file systems and applications to access block devices; downward, it abstracts various heterogeneous disk devices into a unified block device, and will reorder and merge I/O requests from file systems and applications to improve the efficiency of disk access.

Having mastered how disk I/O works, you probably can't wait to find out how you can measure disk I/O performance.

Next, let's take a look at the performance metrics of disks and how to observe them.

## Disk Performance Metrics

When it comes to the measurement of disk performance, it is important to mention five common metrics, which we often use, such as utilization, saturation, IOPS, throughput, and response time. These five metrics are the basic indicators of disk performance.

- Utilization, which is the percentage of time that a disk spends processing I/O. Excessive utilization (e.g., over 80%) usually means there is a performance bottleneck in disk I/O.
- Saturation, which refers to how busy the disk is processing I/O. A high saturation level means that the disk has a serious performance bottleneck. When saturation is 100%, the disk cannot accept new I/O requests.
- IOPS (Input/Output Per Second), which refers to the number of I/O requests per second.
- Throughput, the size of I/O requests per second. - Response Time, which is the time between an I/O request being sent and a response being received.

Note here that the utilization rate only considers the availability of I/O, not the size of I/O. In other words, when the utilization is 100%, the disk may still accept new I/O requests.

These metrics are probably the ones you always talk about and always bring up when discussing disk performance. However, I would like to emphasize that it is important not to compare a metric in isolation, but to analyze it in combination with the read/write ratio, the type of I/O (random or continuous), and the size of the I/O.

For example, in the database, a large number of small files and other such random read and write scenarios, IOPS can better reflect the overall performance of the system; while in multimedia and other sequential read and write scenarios, throughput is more reflective of the overall performance of the system.

Generally speaking, when we select a server for an application, we need to benchmark the disk I/O performance first so that we can accurately assess whether the disk performance can meet the needs of the application.

In this regard, I recommend using the performance testing tool `fio` to test core metrics such as disk IOPS, throughput, and response time. But as always, be flexible in your selection based on local conditions. When benchmarking, be sure to take care to evaluate the metrics specifically based on the application I/O characteristics.

Of course, this requires you to test the performance of different I/O sizes (typically a number of values in the middle of 512B to 1MB) in various scenarios such as random read, sequential read, random write, and sequential write, respectively.

These metrics, obtained with performance tools, can be used as the basis for subsequent analysis of application performance. In case of performance problems, you can use them as limits of disk performance and thus evaluate disk I/O usage.

Understanding disk performance metrics is only the first step in our I/O performance testing. What's the next step to observe them? Here, I'll introduce you to a few common I/O performance observation methods.

## Disk I/O Observations

The first thing to observe is the usage of each disk.

`iostat` is the most commonly used tool for observing disk I/O performance, and it provides a variety of common performance metrics such as per-disk utilization, IOPS, throughput, etc. Of course, these metrics actually come from <ins>/proc/diskstats</ins>.

The output interface of iostat is as follows.

```shell
# -d -x表示显示所有磁盘I/O的指标
$ iostat -d -x 1 
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util 
loop0            0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
loop1            0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sda              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sdb              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00
```

As you can see here, iostat provides a very rich set of performance metrics. The Device in the first column indicates the name of the disk device, and the other columns of metrics, although more in number, are each important in terms of what they mean. To make it easier for you to understand, I have summarized them into a table.

<center><a href="https://ibb.co/8mTmdXP">
    <img src="https://i.ibb.co/FhNh0n8/Interpretation-Of-Iostat-Indicators.png" alt="Interpretation-Of-Iostat-Indicators" border="0">
</a></center>

Among these metrics, you should note:

- %util , which is the disk I/O usage we mentioned earlier;
- r/s+ w/s , which is the IOPS;
- rkB/s+wkB/s , which is the throughput;
- r_await+w_await , which is the response time.

When looking at the metrics, do not forget to analyze them together with the size of the requests (rareq-sz and wareq-sz).

You may notice that disk saturation is not directly available from `iostat`. In fact, there is usually no other easy way to observe saturation, but you can compare the observed, average request queue length or read/write request completion wait time with the results of a benchmark test (e.g., via `fio`) to evaluate the disk saturation in aggregate.

## Process I/O Observations

In addition to per-disk I/O, per-process I/O is also a key concern.

The `iostat` mentioned above only provides overall disk I/O performance data, but the downside is that it does not know which processes are doing disk reads and writes. To see the I/O of the processes, you can also use the tools `pidstat` and `iotop`.

`pidstat` is an old friend of ours, so I won't bore you with its capabilities here. By adding the -d parameter to it, you can see the I/O status of the process as follows:

```shell
$ pidstat -d 1 
13:39:51      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command 
13:39:52      102       916      0.00      4.00      0.00       0  rsyslogd
```

As you can see from the output of pidstat, it provides a real-time view of I/O for each process, including the following

- User ID (UID) and Process ID (PID).
- The size of data read per second (kB_rd/s) in KB.
- The size of write request data issued per second (kB_wr/s) in KB.
- The size of write request data cancelled per second (kB_ccwr/s) in KB.
- The block I/O delay (iodelay), including the time to wait for synchronous block I/O and swap-in block I/O to finish, in clock period.

In addition to the real-time view with `pidstat`, sorting processes by I/O size is also a common method in performance analysis. For this, I recommend another tool, `iotop`. It is a tool similar to `top` that allows you to sort processes by I/O size and find those with larger I/Os.

The output of `iotop` looks like this:

```shell
$ iotop
Total DISK READ :       0.00 B/s | Total DISK WRITE :       7.85 K/s 
Actual DISK READ:       0.00 B/s | Actual DISK WRITE:       0.00 B/s 
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND 
15055 be/3 root        0.00 B/s    7.85 K/s  0.00 %  0.00 % systemd-journald
```

From this output, you can see that the first two lines indicate, respectively, the total number of disk read and write sizes for the process and the total number of real disk read and write sizes. They may not be equal due to cache, buffer, I/O merge, etc.

The rest of the lines represent the I/O of the process from various perspectives, including thread ID, I/O priority, disk read size per second, disk write size per second, and the percentage of clocks swapped in and waiting for I/O.

These are the two tools that we use most often when analyzing disk I/O performance. You first understand their functions and metrics meaning, the specific use of the next case in practice we learn together.

## Summary

Today, we've combed through Linux disk I/O performance metrics and performance tools. We usually use several metrics such as IOPS, throughput, utilization, saturation, and response time to evaluate disk I/O performance.

You can use `iostat` to get a picture of disk I/O, or you can use `pidstat`, `iotop`, etc. to observe process I/O. However, when analyzing these performance metrics, you should be careful to combine them with read/write ratios, I/O types, and I/O sizes.
