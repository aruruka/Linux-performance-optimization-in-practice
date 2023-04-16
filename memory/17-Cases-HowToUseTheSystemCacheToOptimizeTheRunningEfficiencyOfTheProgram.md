# 17 Cases: How to use the system cache to optimize the running efficiency of the program?

In the previous section, we learned the concepts of Buffer and Cache in memory
performance. To briefly review, the design purpose of Buffer and Cache is to
improve the I/O performance of the system. They use memory and act as a bridge
between slow disks and fast CPUs, speeding up I/O access.

Buffer and Cache respectively cache the read and write data to disk and file system.

- From a writing point of view, it not only optimizes the writing of disks and files, but also benefits the application. The application can return to do other work before the data is actually placed on the disk.
- From the perspective of reading, it can not only improve the reading speed of frequently accessed data, but also reduce the pressure of frequent I/O on the disk.

Since Buffer and Cache have a great impact on system performance, can we use
this in the process of software development to optimize I/O performance and improve
the operating efficiency of applications?

The answer is of course yes. Today, I will use a few cases to help you better
understand the role of caches, and learn how to make full use of these caches to
improve program efficiency.

For your convenience, I still express Buffer and Cache in English to avoid confusion
with the word "cache". The "cache" in this article generally refers to the temporary
storage of data in memory.

## Cache hit ratio

Before starting a case, you should habitually ask yourself a question, you want
to accomplish something, how should the result be evaluated? For example, if we
want to use the cache to improve the running efficiency of the program, how should
we evaluate this effect? In other words, is there any metric that can measure the
quality of cache usage?

I guess you have already thought of it, the **cache hit rate**. The so-called cache
hit rate refers to the number of requests to obtain data directly through the
cache, accounting for the percentage of all data requests.

**The higher the hit rate, the higher the benefits of using the cache, and the
better the performance of the application**.

In fact, cache is a core module necessary for all high-concurrency systems today.
Its main function is to read frequently accessed data (that is, hot data) into
memory in advance. In this way, the data can be read directly from the memory
during the next access without going through the hard disk, thereby speeding up
the response speed of the application.

These independent cache modules usually provide a query interface, so that we
can check the cache hit status at any time. However, these interfaces are not
directly provided in the Linux system, so here I will introduce `cachestat` and
`cachetop`, which are the tools to check the system cache hit situation.

- `cachestat` provides read and write hits for the entire operating system cache.
- `cachetop` provides per-process cache hits.

These two tools are part of the 🔗[bcc](https://github.com/iovisor/bcc) package. They are based on the eBPF
(extended Berkeley Packet Filters) mechanism of the Linux kernel to track the
caches managed in the kernel and output the usage and hits of the caches.

Note here that the working principle of eBPF is not our focus today, just remember
the name, and we will study it in detail in later articles. The key point to master
today is how to use these two tools.

Before using `cachestat` and `cachetop`, we first need to install the `bcc` package.
For example, on an Ubuntu system, you can run the following command to install:

```shell
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 4052245BD4284CDD
echo "deb https://repo.iovisor.org/apt/xenial xenial main" | sudo tee /etc/apt/sources.list.d/iovisor.list
sudo apt-get update
sudo apt-get install -y bcc-tools libbcc-examples linux-headers-$(uname -r)
```

> Note: bcc-tools requires kernel version 4.1 or later. If you are using CentOS,
> you need to 🔗[manually upgrade the kernel version before installing](https://github.com/iovisor/bcc/issues/462).

After these steps, all the tools provided by bcc will be installed in the
<ins>/usr/share/bcc/tools</ins> directory. But here’s a reminder that the `bcc`
package does not configure these tools into the system’s PATH by default, so you
have to configure them manually:

```shell
$ export PATH=$PATH:/usr/share/bcc/tools
```

Once configured, you can run the `cachestat` and `cachetop` commands. For example,
the following is a `cachestat` running interface, which outputs 3 sets of cache
statistics at 1-second intervals:

```shell
$ cachestat 1 3
   TOTAL   MISSES     HITS  DIRTIES   BUFFERS_MB  CACHED_MB
       2        0        2        1           17        279
       2        0        2        1           17        279
       2        0        2        1           17        279
```

As you can see, the output of `cachestat` is actually a table. Each row represents
a set of data, and each column represents a different cache statistic. These
indicators represent, from left to right:

- TOTAL , indicating the total number of I/O times;
- MISSES , indicating the number of cache misses;
- HITS , indicating the number of cache hits;
- DIRTIES, indicating the number of dirty pages added to the cache;
- BUFFERS_MB indicates the size of Buffers, in MB;
- CACHED_MB indicates the size of the Cache in MB.

Next, let's look at a `cachetop` running interface:

```shell
$ cachetop
11:58:50 Buffers MB: 258 / Cached MB: 347 / Sort: HITS / Order: ascending
PID      UID      CMD              HITS     MISSES   DIRTIES  READ_HIT%  WRITE_HIT%
   13029 root     python                  1        0        0     100.0%       0.0%
```

Its output is similar to `top`, sorted by cache hits (HITS) by default, showing
the cache hits of each process. Specific to each indicator, HITS, MISSES and DIRTIES
here have the same meaning as in cachestat, which respectively represent the number
of cache hits, the number of misses, and the number of dirty pages added to the
cache within the interval.

And READ_HIT and WRITE_HIT represent the cache hit rate of read and write respectively.

## Specifies the cache size of the file

In addition to the cache hit rate, there is another metric you may be interested
in, which is the cache size of the specified file in memory. You can use the tool
🔗[pcstat](https://github.com/tobert/pcstat) to view the cache size and cache ratio of files in memory.

`pcstat` is a tool developed based on Go language, so before installing it, you
should first install Go language, you can click 🔗[here](https://golang.org/dl/) to download and install.

After installing the Go language, run the following command to install `pcstat`:

```shell
$ export GOPATH=~/go
$ export PATH=~/go/bin:$PATH
$ go get golang.org/x/sys/unix
$ go get github.com/tobert/pcstat/pcstat
```

After everything is installed, you can run `pcstat` to view the file cache. For
example, here is an example of a `pcstat` run showing the cache of the
<ins>/bin/ls</ins> file:

```shell
$ pcstat /bin/ls
+---------+----------------+------------+-----------+---------+
| Name    | Size (bytes)   | Pages      | Cached    | Percent |
|---------+----------------+------------+-----------+---------|
| /bin/ls | 133792         | 33         | 0         | 000.000 |
+---------+----------------+------------+-----------+---------+
```

In this output, Cached is the size of <ins>/bin/ls</ins> in the cache, and
Percent is the percentage of the cache. You see they are all 0, which means
<ins>/bin/ls</ins> is not in the cache.

Then, if you execute the `ls` command and then run the same command to check, you
will find that <ins>/bin/ls</ins> is in the cache:

```shell
$ ls
$ pcstat /bin/ls
+---------+----------------+------------+-----------+---------+
| Name    | Size (bytes)   | Pages      | Cached    | Percent |
|---------+----------------+------------+-----------+---------|
| /bin/ls | 133792         | 33         | 33        | 100.000 |
+---------+----------------+------------+-----------+---------+
```

After knowing the corresponding indicators of the cache and the method of viewing
the system cache, next, we will enter today's formal case.

Like the previous case, today's case is also based on Ubuntu 18.04, of course it
is also applicable to other Linux systems.

- Machine configuration: 2 CPU, 8GB memory.
- Install the `bcc` and `pcstat` packages in advance according to the above steps, and add the installation path of these tools to the PATH environment variable.
- pre-installed Docker packages, e.g. `apt-get install docker.io`

## Case 1

In the first case, let's take a look at the `dd` command mentioned in the previous
lecture.

As a tool for copying disks and files, `dd` is often used to test the read and
write performance of disks or file systems. However, since caching affects
performance, what happens to the test results if you use `dd` to perform multiple
read tests on the same file?

Let's try it out. First, open two terminals, connect to the Ubuntu machine, and
make sure that `bcc` has been installed and configured successfully.

Then, use the `dd` command to generate a temporary file for subsequent file reading tests:

```shell
# 生成一个512MB的临时文件
$ dd if=/dev/sda1 of=file bs=1M count=512
# 清理缓存
$ echo 3 > /proc/sys/vm/drop_caches
```

Continuing in the first terminal, run the `pcstat` command to verify that the
file just generated is not in the cache. If all is well, you'll see that both
Cached and Percent are 0:

```shell
$ pcstat file
+-------+----------------+------------+-----------+---------+
| Name  | Size (bytes)   | Pages      | Cached    | Percent |
|-------+----------------+------------+-----------+---------|
| file  | 536870912      | 131072     | 0         | 000.000 |
+-------+----------------+------------+-----------+---------+
```

Still in the first terminal, now run the `cachetop` command:

```shell
# 每隔5秒刷新一次数据
$ cachetop 5
```

This time in the second terminal, run the `dd` command to test the reading
speed of the file:

```shell
$ dd if=file of=/dev/null bs=1M
512+0 records in
512+0 records out
536870912 bytes (537 MB, 512 MiB) copied, 16.0509 s, 33.4 MB/s
```

As can be seen from the results of `dd`, the read performance of this file is
33.4 MB/s. Since we have cleared the cache before the `dd` command runs, when
the `dd` command reads data, it must be read from the disk through the file system.

However, does this mean that all read requests of `dd` can be sent directly to
the disk?

Let's go back to the first terminal and check the cache hit status of the
cachetop interface:

```shell
PID      UID      CMD              HITS     MISSES   DIRTIES  READ_HIT%  WRITE_HIT%
\.\.\.
    3264 root     dd                  37077    37330        0      49.8%      50.2%
```

From the results of `cachetop`, it can be found that not all reads fall to the disk.
In fact, the cache hit rate of read requests is only 50%.

Next, we proceed to try the same test command. Switch to the second terminal first,
and execute the `dd` command just now again:

```shell
$ dd if=file of=/dev/null bs=1M
512+0 records in
512+0 records out
536870912 bytes (537 MB, 512 MiB) copied, 0.118415 s, 4.5 GB/s
```

Are you a little surprised by the results? The read performance of the disk has
actually become 4.5 GB/s, which is significantly higher than the first result.
Why are the results so good this time?

Let’s go back to the first terminal and see what’s going on with `cachetop`:

```shell
10:45:22 Buffers MB: 4 / Cached MB: 719 / Sort: HITS / Order: ascending
PID      UID      CMD              HITS     MISSES   DIRTIES  READ_HIT%  WRITE_HIT%
\.\.\.
   32642 root     dd                 131637        0        0     100.0%       0.0%
```

Obviously, `cachetop` has also changed a lot. You can find that the cache hit rate
of this read is 100.0%, that is to say, all the `dd` commands this time hit the cache,
so you can see such high performance.

Then, go back to the second terminal and execute `pcstat` again to check the cache
status of the file file:

```shell
$ pcstat file
+-------+----------------+------------+-----------+---------+
| Name  | Size (bytes)   | Pages      | Cached    | Percent |
|-------+----------------+------------+-----------+---------|
| file  | 536870912      | 131072     | 131072    | 100.000 |
+-------+----------------+------------+-----------+---------+
```

From the results of `pcstat`, you can find that the test file file has been fully
cached, which is consistent with the 100% cache hit rate observed just now.

These two results show that the system cache can significantly accelerate the
second `dd` operation, which can greatly improve the performance of file reading.

But at the same time, it should be noted that if we use `dd` as a tool to test the
performance of the file system, the test results will be seriously distorted due
to the existence of the cache.

## Case 2

Next, let's look at a case of file reading and writing. This case is similar to
the example of the uninterruptible state process learned earlier. Its basic
function is relatively simple, that is, it reads 32MB of data from the disk partition
<ins>/dev/sda1</ins> per second, and prints out the time it takes to read the data.

For your convenience, I packaged it into a 🔗[Docker image](https://github.com/feiskyer/linux-perf-examples/tree/master/io-cached). Similar to the
previous case, I provide the following two options. You can adjust the path of
the disk partition and the size of I/O according to the system configuration.

- -d option, set the disk or partition path to read, the default is to find the disk with the prefix /dev/sd or /dev/xvd.
- -s option, set the amount of data read each time, in bytes, the default is 33554432 (that is, 32MB).

This case also requires you to open two terminals. After logging in to the machine
with SSH, first run the `cachetop` command in the first terminal:

```shell
# 每隔5秒刷新一次数据
$ cachetop 5
```

Then, go to the second terminal and execute the following command to run the case:

```shell
$ docker run --privileged --name=app -itd feisky/app:io-direct
```

After the case is running, we also need to run the following command to confirm
that the case has started normally. If everything went well, you should see output
similar to the following:

```shell
$ docker logs app
Reading data from disk /dev/sdb1 with buffer size 33554432
Time used: 0.929935 s to read 33554432 bytes
Time used: 0.949625 s to read 33554432 bytes
```

From here you can see that it takes 0.9 seconds to read every 32 MB of data. Is
this time reasonable? I think your first reaction is, it's too slow. So is this
caused by not using the system cache?

Let's check again. Go back to the first terminal and look at the output of `cachetop`.
Here, we find the cache usage of the case process app:

```shell
16:39:18 Buffers MB: 73 / Cached MB: 281 / Sort: HITS / Order: ascending
PID      UID      CMD              HITS     MISSES   DIRTIES  READ_HIT%  WRITE_HIT%
   21881 root     app                  1024        0        0     100.0%       0.0% 
```

This output seems a bit interesting. All 1024 cache hits, and the read hit rate
is 100%. It seems that all read requests have passed through the system cache.
But here comes the problem again. If it is really all cached I/O, the reading
speed should not be so slow.

However, having said that, we seem to have overlooked another important factor,
the actual size of the data read per second. HITS represents the number of cache
hits, so how much data can be read for each hit? Naturally it is one page.

As mentioned earlier, memory is managed in units of pages, and the size of each
page is 4KB. Therefore, in the time interval of 5 seconds, the hit cache is
$1024*4K/1024 = 4MB$, and then divided by 5 seconds, the cache read per second
is 0.8MB, which is obviously far from the 32 MB/s of the case application.

As for why you can only see 0.8 MB HITS, we will explain later, here you just
need to know how to analyze according to the results.

This further verifies our conjecture that the system cache is not fully utilized
in this case. In fact, we have encountered a similar problem before.
If the direct I/O flag is set for the system call, the system cache can be bypassed.

Well, the easiest way to tell if an application is using direct I/O is of course
to observe its system calls and find out what options the application has when
calling them. What tools are used to observe system calls? Naturally, `strace`.

Continue to run the following `strace` command in Terminal 2 to observe the system
calls of the case application. Note that the `pgrep` command is used here to
find the PID number of the case process:

```shell
# strace -p $(pgrep app)
strace: Process 4988 attached
restart_syscall(<\.\.\. resuming interrupted nanosleep \.\.\.>) = 0
openat(AT_FDCWD, "/dev/sdb1", O_RDONLY|O_DIRECT) = 4
mmap(NULL, 33558528, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f448d240000
read(4, "8vq\213\314\264u\373\4\336K\224\25@\371\1\252\2\262\252q\221\n0\30\225bD\252\266@J"\.\.\., 33554432) = 33554432
write(1, "Time used: 0.948897 s to read 33"\.\.\., 45) = 45
close(4)                                = 0
```

From the results of `strace`, we can see that the case application calls openat to
open the disk partition <ins>/dev/sdb1</ins>, and the incoming parameter is
O_RDONLY|O_DIRECT (the vertical bar in the middle indicates or).

O_RDONLY means open for read-only, while O_DIRECT means open for direct read,
which bypasses the system's cache.

After verifying this, it is easy to understand why it takes so long to read 32 MB
of data. The speed of reading and writing directly from the disk is naturally
much slower than reading and writing to the cache. This is also the greatest
significance of the existence of the cache.

After finding out the problem, we can also look at the 🔗[source code](https://github.com/feiskyer/linux-perf-examples/blob/master/io-cached/app.c) of
the case application and verify it again:

```c
int flags = O_RDONLY | O_LARGEFILE | O_DIRECT; 
int fd = open(disk, flags, 0755);
```

The above code clearly tells us: it really uses direct I/O.

After finding out the reason for the slow disk reading, it is natural to optimize
the performance of disk reading. Modifying the source code, removing the O_DIRECT
option, and letting the application use cached I/O instead of direct I/O can speed
up disk reads.

🔗[app-cached.c](https://github.com/feiskyer/linux-perf-examples/blob/master/io-cached/app-cached.c) is the repaired source code, and I also packaged it into a
container image. In the second terminal, press Ctrl+C to stop the previous `strace`
command, run the following command, you can start it:

```shell
# 删除上述案例应用
$ docker rm -f app

# 运行修复后的应用
$ docker run --privileged --name=app -itd feisky/app:io-cached
```

Still in the second terminal, run the following command to view the log of the
new application, you should see the following output:

```shell
$ docker logs app
Reading data from disk /dev/sdb1 with buffer size 33554432
Time used: 0.037342 s s to read 33554432 bytes
Time used: 0.029676 s to read 33554432 bytes
```

Let's go back to the first terminal and check the output of `cachetop` to confirm:

```shell
16:40:08 Buffers MB: 73 / Cached MB: 281 / Sort: HITS / Order: ascending
PID      UID      CMD              HITS     MISSES   DIRTIES  READ_HIT%  WRITE_HIT%
   22106 root     app                 40960        0        0     100.0%       0.0%
```

Sure enough, the hit rate of reading is still 100%, but HITS (that is, the number of hits)
has become 40960. Calculate it in the same way, and the number of bytes per second
is exactly 32 MB (that is, $40960*4k/5/1024=32M$).

This case shows that fully utilizing the system cache can greatly improve performance
when performing I/O operations. However, when observing the cache hit rate, we
should also pay attention to the actual I/O size of the application and comprehensively
analyze the usage of the cache.

At the end of the case, let’s go back to the first question. Why can `cachetop`
only see all hits of a small part of data before optimization, but no misses for
a large amount of data? This is because the `cachetop` tool does not count direct
I/O. This also shows once again that it is important to understand the principles
of tools.

> The calculation method of `cachetop` involves the principle of I/O and some kernel
> knowledge. If you want to understand its principle, you can click 🔗[here](https://github.com/iovisor/bcc/blob/master/tools/cachetop.py)
> to view its source code.

## Summary

Buffers and Cache can greatly improve the I/O performance of the system. Usually,
we use cache hit rate to measure the efficiency of cache usage. The higher the
hit rate, the better the cache is being utilized and the better the performance
of the application.

You can use the tools `cachestat` and `cachetop` to observe the cache hits of
the system and process. in,

- `cachestat` provides read and write hits for the entire system cache.
- cachetop provides per-process cache hits.

However, it should be noted that both Buffers and Cache are managed by the
operating system, and the application cannot directly control the content and
life cycle of these caches. Therefore, in application development, it is generally
necessary to use a dedicated cache component to further improve performance.

For example, the program can use the heap or stack to explicitly declare the memory
space to store the data that needs to be cached. Or, use an external cache service
such as Redis to optimize data access efficiency.