# 31 Routines: Thinkings of Disk I/O performance tuning

In the last section, we reviewed common file system and disk I/O performance metrics, sorted out core I/O performance observation tools, and finally summarized ideas for quickly analyzing I/O performance problems.

Although there are many I/O performance metrics and several corresponding performance analysis tools, once you understand the meaning of various metrics, you will find that they are all related in some way.

Following these relationships down, you will find that it is not difficult to master these common bottleneck analysis ideas.

After finding out the I/O performance bottleneck, the next step is to optimize it, that is, how to complete I/O operations at the fastest speed, or change the idea to reduce or even avoid disk I/O operations.

Today, I'll talk about ideas and considerations for optimizing I/O performance issues.

## I/O benchmarking

As is my habit, before I optimize, I ask myself, what is the goal of I/O performance optimization? In other words, what are the appropriate I/O performance metrics (e.g., IOPS, throughput, latency, etc.) that we should observe?

In fact, the exact criteria for I/O performance metrics are expected to be answered differently by each person, as each of us may have different application scenarios, file systems and physical disks used, etc.

In order to evaluate the optimization effect more objectively and reasonably, we should first benchmark the disk and file system to get the limit performance of file system or disk I/O.

🔗[fio](https://github.com/axboe/fio) (Flexible I/O Tester) is exactly the most commonly used file system and disk I/O performance benchmarking tool. It provides a large number of customizable options to test the I/O performance of bare disks or file systems in a variety of scenarios, including different block sizes, different I/O engines, and with or without caching.

The installation of `fio` is relatively simple, you can install it by executing the following command:

```shell
# Ubuntu
apt-get install -y fio

# CentOS
yum install -y fio
```

Once the installation is complete, you can run `man fio` to query how it works. There are many options for `fio`, so I will go through a few common scenarios and introduce some of the most commonly used options. These common scenarios include random reads, random writes, sequential reads, and sequential writes, and you can test them by executing the following commands:

```shell
# 随机读
fio -name=randread -direct=1 -iodepth=64 -rw=randread -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb

# 随机写
fio -name=randwrite -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb

# 顺序读
fio -name=read -direct=1 -iodepth=64 -rw=read -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb

# 顺序写
fio -name=write -direct=1 -iodepth=64 -rw=write -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb
```

In this, there are a few parameters that you need to focus on.

- direct, indicates whether to skip the system cache. In the example above, I set 1 to skip the system cache.
- iodepth, which indicates the maximum number of simultaneous I/O requests when using asynchronous I/O (AIO for short). In the example above, I set it to 64.
- rw, which indicates the I/O mode. In my example, read/write means sequential read/write respectively, while randread/randwrite means random read/write respectively.
- ioengine, the I/O engine, supports various I/O engines such as synchronous (sync), asynchronous (libaio), memory mapping (mmap), and network (net). In the above example, I set libaio to indicate the use of asynchronous I/O.
- bs, the size of I/O. In the example, I set it to 4K (which is also the default value).
- filename, the path to the file. Of course, it can be either a disk path (to test disk performance) or a file path (to test file system performance). In the example, I set it to disk <ins>/dev/sdb</ins>. Note, however, that writing with a disk path test will destroy the file system in this disk, so you must make a backup of your data beforehand before using it.

Here is an example of a report that I read using the `fio` test sequence.

```plaintext
read: (g=0): rw=read, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=64
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [R(1)][100.0%][r=16.7MiB/s,w=0KiB/s][r=4280,w=0 IOPS][eta 00m:00s]
read: (groupid=0, jobs=1): err= 0: pid=17966: Sun Dec 30 08:31:48 2018
   read: IOPS=4257, BW=16.6MiB/s (17.4MB/s)(1024MiB/61568msec)
    slat (usec): min=2, max=2566, avg= 4.29, stdev=21.76
    clat (usec): min=228, max=407360, avg=15024.30, stdev=20524.39
     lat (usec): min=243, max=407363, avg=15029.12, stdev=20524.26
    clat percentiles (usec):
     |  1.00th=[   498],  5.00th=[  1020], 10.00th=[  1319], 20.00th=[  1713],
     | 30.00th=[  1991], 40.00th=[  2212], 50.00th=[  2540], 60.00th=[  2933],
     | 70.00th=[  5407], 80.00th=[ 44303], 90.00th=[ 45351], 95.00th=[ 45876],
     | 99.00th=[ 46924], 99.50th=[ 46924], 99.90th=[ 48497], 99.95th=[ 49021],
     | 99.99th=[404751]
   bw (  KiB/s): min= 8208, max=18832, per=99.85%, avg=17005.35, stdev=998.94, samples=123
   iops        : min= 2052, max= 4708, avg=4251.30, stdev=249.74, samples=123
  lat (usec)   : 250=0.01%, 500=1.03%, 750=1.69%, 1000=2.07%
  lat (msec)   : 2=25.64%, 4=37.58%, 10=2.08%, 20=0.02%, 50=29.86%
  lat (msec)   : 100=0.01%, 500=0.02%
  cpu          : usr=1.02%, sys=2.97%, ctx=33312, majf=0, minf=75
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued rwt: total=262144,0,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=64

Run status group 0 (all jobs):
   READ: bw=16.6MiB/s (17.4MB/s), 16.6MiB/s-16.6MiB/s (17.4MB/s-17.4MB/s), io=1024MiB (1074MB), run=61568-61568msec

Disk stats (read/write):
  sdb: ios=261897/0, merge=0/0, ticks=3912108/0, in_queue=3474336, util=90.09% 
```

In this report, we need to focus on the lines slat, clat, lat, and bw and iops.

Let's look at the first three parameters just mentioned. In fact, slat, clat, and lat all refer to I/O latency. The differences are:

- slat , the time from I/O commit to actual I/O execution (Submission latency);
- clat , the time from I/O commit to I/O completion (Completion latency);
- and lat , the total time from I/O creation by fio to I/O completion.

Note that for synchronous I/O, since I/O commit and I/O completion are one action, slat is actually the I/O completion time and clat is 0. As you can see from the example, when using asynchronous I/O (libaio), lat is approximately equal to the sum of slat + clat.

Let's look at bw, which represents throughput. In my example above, you can see that the average throughput is about 16 MB (17005 KiB/1024).

The last iops is actually the number of I/Os per second, and the average IOPS in the example above is 4250.

Typically, application I/O is read and write in parallel, and the I/O size is not necessarily the same each time. Therefore, the scenarios just described do not accurately simulate the I/O patterns of the application. So how can we accurately simulate the I/O pattern of an application?

Fortunately, `fio` supports replay of I/O. With `blktrace`, as mentioned earlier, and `fio`, you can benchmark the I/O patterns of your application. You need to first use `blktrace` to record I/O accesses to disk devices, and then use `fio` to replay the `blktrace` record.

For example, you could run the following command to do this:

```shell
# 使用blktrace跟踪磁盘I/O，注意指定应用程序正在操作的磁盘
$ blktrace /dev/sdb

# 查看blktrace记录的结果
# ls
sdb.blktrace.0  sdb.blktrace.1

# 将结果转化为二进制文件
$ blkparse sdb -d sdb.bin

# 使用fio重放日志
$ fio --name=replay --filename=/dev/sdb --direct=1 --read_iolog=sdb.bin
```

This way, we get a benchmark report of the application I/O mode by using the combination of `blktrace` + `fio`.

## I/O performance tunning

Once you have the I/O benchmarking report, you can use the performance analysis routine we summarized in the previous section to identify I/O performance bottlenecks and optimize them. Of course, you can't optimize I/O performance without the help of the Linux I/O stack diagram. You can review it in the context of the following I/O stack diagram.

<center><a href="https://imgbb.com/">
   <img src="https://i.ibb.co/3fGP12X/Architecture-Diagram-Of-Linux-File-System.png" alt="Architecture-Diagram-Of-Linux-File-System" border="0">
</a></center>

Below, I'll take you through the basic ideas of I/O performance optimization from the application, file system, and disk perspectives, respectively.

### Application tunning

First, let's look at the idea of optimizing I/O from the application's perspective.

The application is at the top of the I/O stack, it can tune the I/O mode (sequential or random, synchronous or asynchronous) through system calls, and it is also the ultimate source of I/O data. In my opinion, there are several ways to optimize the I/O performance of an application.

First, you can replace random writes with append writes to reduce addressing overhead and speed up I/O writes.

Second, you can use cached I/O to take advantage of the system cache and reduce the number of actual I/Os.

Third, you can build your own cache inside the application or use an external caching system such as Redis. This way, on the one hand, you can control the data and life cycle of the cache within the application, and on the other hand, you can reduce the impact of other applications using the cache on yourself.

For example, we've seen in the MySQL case earlier that just because an interfering application clears the system cache, it can cause hundreds of times the performance difference in MySQL queries (0.1s vs. 15s).

Then again, library functions such as `fopen` and `fread` provided by the C standard library make use of the standard library's cache to reduce disk operations. When you use `open`, `read` and other system calls directly, you can only use the page cache and buffers provided by the operating system, and there is no cache available for the library functions.

Fourth, when you need to read and write frequently to the same disk space, you can use `mmap` instead of `read`/`write` to reduce the number of memory copies.

Fifth, in scenarios that require synchronous writes, try to combine write requests instead of having each request written to disk synchronously, i.e., you can replace `O_SYNC` with `fsync()`.

Sixth, when multiple applications share the same disk, it is recommended that you use the I/O subsystem of cgroups to limit the IOPS and throughput of processes/process groups in order to ensure that I/O is not fully occupied by one application.

Finally, when using the CFQ scheduler, you can use ionice to adjust the I/O scheduling priority of processes, especially to increase the I/O priority of core applications. ionice supports three priority classes: Idle, Best-effort, and Realtime. ionice also supports levels 0-7 for Best-effort and Realtime, respectively. The smaller the value, the higher the priority.

### File system tunning

When an application accesses a normal file, the file system is actually indirectly responsible for reading and writing the file to and from the disk. So, there are many ways to optimize I/O performance related to the file system as well.

First, you can choose the most suitable filesystem depending on the actual load scenario. For example, Ubuntu uses ext4 filesystem by default, while CentOS 7 uses xfs filesystem by default.

Compared to ext4, xfs supports larger disk partitions and larger number of files, e.g. xfs supports disks larger than 16TB. However, the downside of the xfs file system is that it cannot be shrunk, whereas ext4 can.

Second, after selecting a file system, you can further optimize the file system configuration options, including file system characteristics (e.g. ext_attr, dir_index), logging modes (e.g. journal, ordered, writeback), mounting options (e.g. noatime), and so on.

For example, the `tune2fs` tool can be used to tune the file system characteristics (`tune2fs` is also commonly used to view the contents of file system superblocks). And with <ins>/etc/fstab</ins>, or the mount command line parameter, we can tune the file system's logging mode, mount options, etc.

Third, you can optimize the file system cache.

For example, you can optimize the pdflush dirty page refresh frequency (such as setting dirty_expire_centisecs and dirty_writeback_centisecs) and dirty page limit (such as adjusting dirty_background_ratio and dirty_ratio, etc.).

Then again, you can also optimize the kernel's tendency to reclaim directory entry caches and index node caches by adjusting vfs_cache_pressure (<ins>/proc/sys/vm/vfs_cache_pressure</ins>, default value 100), with larger values indicating easier reclaim.

Finally, you can also use the in-memory file system tmpfs for better I/O performance when persistence is not needed. tmpfs keeps data directly in memory instead of on disk. For example, <ins>/dev/shm/</ins>, is an in-memory file system configured by default for most Linux, and it defaults to half the size of total memory.

### Disk tunning

Persistent storage of data ultimately comes down to specific physical disks, and at the same time, disks are the bottom layer of the entire I/O stack. From the disk perspective, there are naturally many effective ways to optimize performance.

First, the simplest and most effective optimization method is to switch to a better performing disk, such as replacing HDDs with SSDs.

Second, we can use RAID to combine multiple disks into one logical disk to form a redundant array of independent disks. This can improve both data reliability and data access performance.

Third, we can choose the most suitable I/O scheduling algorithm for the characteristics of disk and application I/O patterns. For example, for SSDs and disks in virtual machines, the noop scheduling algorithm is usually used. For database applications, I recommend using the deadline algorithm.

Fourth, we can isolate the application's data at the disk level. For example, we can configure separate disks for logs, databases, and other applications that are under heavy I/O pressure.

Fifth, in a more sequential read scenario, we can increase the pre-reading data on disk. For example, you can adjust the pre-reading size of <ins>/dev/sdb</ins> in the following two ways.

- Adjusting the kernel option <ins>/sys/block/sdb/queue/read_ahead_kb</ins>, where the default size is 128 KB in KB. - Using the `blockdev` utility to set, for example, `blockdev --setra 8192 /dev/sdb`, note that the units here are 512B (0.5KB), so it's twice the total value of read_ahead_kb.

Sixth, we can optimize the options for kernel block device I/O. For example, you can adjust the length of the disk queue <ins>/sys/block/sdb/queue/nr_requests</ins>. Increasing the queue length appropriately can improve the disk throughput (and of course lead to higher I/O latency).

Finally, note that a hardware error on the disk itself can also cause a sharp drop in I/O performance, so when you find a sharp drop in disk performance, you also need to confirm that there is a hardware error on the disk itself.

For example, you can check `dmesg` for a log of hardware I/O errors. You can also use tools like `badblocks`, `smartctl`, etc. to detect hardware problems on the disk, or `e2fsck`, etc. to detect file system errors. If you find a problem, you can use tools like `fsck` to fix it.

## Summary

Today, we combed through common file system and disk I/O performance optimization ideas and methods together. When you find I/O performance problems, don't rush to optimize them. Instead, identify the most important issues that can maximize performance, and then consider specific optimization methods starting from different layers of the I/O stack.

Remember that disk and file system I/O is usually the slowest module in the entire system. So, when optimizing I/O problems, you can reduce I/O calls with the help of faster memory, network, CPU, etc., in addition to optimizing the I/O execution flow.

For example, you can take advantage of the Buffer, Cache, or internal cache provided by the system, or an external caching system such as Redis.
