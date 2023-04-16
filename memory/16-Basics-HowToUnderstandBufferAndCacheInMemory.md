# 16 Basics: How To Understand Buffer And Cache In Memory?

In the previous section, we sorted out the basic principles of Linux memory management,
and learned to use tools such as `free` and `top` to view the memory usage of the
system and processes.

The relationship between memory and CPU is very close, and memory management
itself is also a very complicated mechanism, so it is normal to feel that knowledge
is very hard-core and difficult to chew. But still the same sentence, you don’t
have to understand all the content when you are a beginner. It is often much easier
to go back and review after you continue to learn, understand more related concepts
and cooperate with certain practices. Of course, basic skills cannot be given up.

Before starting today's content, let's review the memory usage of the system, such
as the following free output interface:

```shell
# 注意不同版本的free输出可能会有所不同
$ free
              total        used        free      shared  buff/cache   available
Mem:        8169348      263524     6875352         668     1030472     7611064
Swap:             0           0           0
```

Obviously, this interface includes the specific usage of physical memory Mem and
swap partition Swap, such as total memory, used memory, cache, available memory,
etc. Among them, cache is the sum of Buffer and Cache.

Most of the indicators here are relatively easy to understand, but Buffer and
Cache may not be easy to distinguish. Literally speaking, Buffer is a buffer, and
Cache is a cache, both of which are temporary storage of data in memory. So, do
you know what is the difference between these two kinds of "temporary storage"?

Note: In the rest of today's content, I will express Buffer and Cache in English
to avoid confusion with the word "cache" in the text. The "cache" in this article
generally refers to temporary storage in memory.

## Source of `free` data

Before I formally explain the two concepts, you can think about it, is there any
way for you to learn more about them? In addition to getting the concept directly
from Chinese translation, don't forget that Buffer and Cache are still the
indicators we get with `free`.

Remember what I said before, what should I do when I encounter an indicator that
I don’t understand?

I guess you remembered it, so if you don’t understand it, check the manual. Use
the `man` command to query the free documentation, and you can find detailed
descriptions of the corresponding indicators. For example, if we execute `man free`,
we can see the following interface.

```plaintext
buffers
              Memory used by kernel buffers (Buffers in /proc/meminfo)

       cache  Memory used by the page cache and slabs (Cached and SReclaimable in /proc/meminfo)

       buff/cache
              Sum of buffers and cache
```

From the manual of `free`, you can see the description of buffer and cache.

- Buffers is the memory used by the kernel buffer, corresponding to the Buffers value in <ins>/proc/meminfo</ins>.
- Cache is the memory used by the kernel page cache and Slab, corresponding to the sum of Cached and SReclaimable in <ins>/proc/meminfo</ins>.

The instructions here tell us that these values are all from <ins>/proc/meminfo</ins>,
but the more specific meanings of Buffers, Cached, and SReclaimable are still unclear.

To find out what they are, I guess your first reaction is to go to Baidu or Google.
Although in most cases, a web search can give an answer. However, not to mention
the time and energy spent screening information, it is difficult for you to
guarantee the accuracy of this answer.

It should be noted that the conclusions on the Internet may be correct, but they
may not match your environment. In the simplest terms, the specific meaning of
the same indicator may vary greatly due to different kernel versions and
performance tool versions. This is why, I always emphasize general ideas and
methods in the column, rather than letting you memorize conclusions. For case
practice, the machine environment is our greatest limitation.

So, is there an easier and more accurate way to query their meanings?

## proc file system

As I mentioned in the previous CPU performance module, <ins>/proc</ins> is a special
file system provided by the Linux kernel, and it is an interface for users to
interact with the kernel. For example, users can query the running status and
configuration options of the kernel from <ins>/proc</ins>, and query the running
status and statistical data of the process. Of course, you can also modify the
configuration of the kernel through <ins>/proc</ins>.

The proc file system is also the ultimate data source for many performance tools.
For example, the `free` we just saw is to get the memory usage by reading
<ins>/proc/meminfo</ins>.

Going back to <ins>/proc/meminfo</ins>, since the indicators of Buffers, Cached,
and SReclaimable are not easy to understand, we have to continue to check the proc
file system to obtain their detailed definitions.

Execute `man proc`, you can get detailed documentation of the proc file system.

Note that this document is relatively long, you'd better search it (for example,
search "meminfo") to locate the memory part faster.

```plaintext
Buffers %lu
    Relatively temporary storage for raw disk blocks that shouldn't get tremendously large (20MB or so).

Cached %lu
   In-memory cache for files read from the disk (the page cache).  Doesn't include SwapCached.
...
SReclaimable %lu (since Linux 2.6.19)
    Part of Slab, that might be reclaimed, such as caches.
    
SUnreclaim %lu (since Linux 2.6.19)
    Part of Slab, that cannot be reclaimed on memory pressure.
```

Through this document, we can see:

- Buffers is the temporary storage of the original disk blocks, that is, the data **used to cache the disk**, usually not particularly large (about 20MB). In this way, the kernel can gather scattered writes and optimize disk writes uniformly. For example, multiple small writes can be combined into a single large write, etc.
- Cached is a page cache that reads files from disk, that is, it is **used to cache data read from files**. In this way, the next time these file data are accessed, they can be quickly obtained directly from the memory without having to access the slow disk again.
- SReclaimable is part of Slab. Slab consists of two parts, the reclaimable part is recorded with SReclaimable; the non-reclaimable part is recorded with SUnreclaim.

Well, we finally found the detailed definitions of these three indicators. At this
point, do you breathe a sigh of relief and think with satisfaction that you finally
figured out Buffer and Cache. However, do you really understand this definition?
Here I ask you two questions, you first think about whether you can answer them.

The first question, Buffer's documentation does not mention whether it is a cache
of disk read data or write data, and many Internet search results will mention
that Buffer is **only a cache of data that will be written to disk**. On the other hand,
will it also cache data read from disk?

The second question, as mentioned in the documentation, Cache is a cache for data
read from files, so will it also cache data written to files?

In order to answer these two questions, next, I will use several cases to show
the usage of Buffer and Cache in different scenarios.

## Cases

### Your preparation

Like the previous experiment, today's case is also based on Ubuntu 18.04, of
course, other Linux systems are also applicable. My case environment is like this.

- Machine configuration: 2 CPU, 8GB memory.
- Pre-install the `sysstat` package, eg `apt install sysstat`.

The reason to install `sysstat` is because we need to use `vmstat` to observe the
changes of Buffer and Cache. Although the same results can be read from
<ins>/proc/meminfo</ins>, the results of `vmstat` are more intuitive after all.

In addition, these cases use dd to simulate disk and file I/O, so we also need
to observe the changes of I/O.

After the above tools are installed, you can open two terminals and connect to
the Ubuntu machine.

The last step in the preparation process, in order to reduce the impact of the cache, remember to run the following command in the first terminal to clear the system cache:

```shell
# 清理文件页、目录项、Inodes等各种缓存
$ echo 3 > /proc/sys/vm/drop_caches
```

The <ins>/proc/sys/vm/drop_caches</ins> here is an example of modifying the kernel
behavior through the proc file system. Writing "3" means cleaning various caches
such as file pages, directory entries, and Inodes. You don't have to worry about
the difference between these types of caches for now, we will talk about them later.

### Scenario 1: Case of disk and file write

Let's simulate the first scenario first. First, in the first terminal, run the
following `vmstat` command:

```shell
# 每隔1秒输出1组数据
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
0  0      0 7743608   1112  92168    0    0     0     0   52  152  0  1 100  0  0
0  0      0 7743608   1112  92168    0    0     0     0   36   92  0  0 100  0  0
```

In the output interface, the buff and cache in the memory part, and the bi and bo
in the io part are the focus of our attention.

- buff and cache are the Buffers and Cache we saw earlier, and the unit is KB.
- bi and bo represent the read and write sizes of the block device respectively, in blocks/second. Since the block size in Linux is 1KB, this unit is equivalent to KB/s.

Under normal circumstances, on an idle system, what you should see is that these
values have remained unchanged for multiple results.

Next, go to the second terminal and execute the `dd` command to generate a 500MB
file by reading a random device:

```shell
$ dd if=/dev/urandom of=/tmp/file bs=1M count=500
```

Then go back to the first terminal and observe the changes of Buffer and Cache:

```shell
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
0  0      0 7499460   1344 230484    0    0     0     0   29  145  0  0 100  0  0
1  0      0 7338088   1752 390512    0    0   488     0   39  558  0 47 53  0  0
1  0      0 7158872   1752 568800    0    0     0     4   30  376  1 50 49  0  0
1  0      0 6980308   1752 747860    0    0     0     0   24  360  0 50 50  0  0
0  0      0 6977448   1752 752072    0    0     0     0   29  138  0  0 100  0  0
0  0      0 6977440   1760 752080    0    0     0   152   42  212  0  1 99  1  0
...
0  1      0 6977216   1768 752104    0    0     4 122880   33  234  0  1 51 49  0
0  1      0 6977440   1768 752108    0    0     0 10240   38  196  0  0 50 50  0
```

By observing the output of `vmstat`, we found that when the `dd` command is running,
the Cache keeps growing, while the Buffer basically remains unchanged.

Looking further at the I/O situation, you'll see,

- When Cache first started to grow, block device I/O was rare, bi only appeared once at 488 KB/s, and bo only once at 4KB. After a period of time, a large number of block device writes will appear, for example, bo becomes 122880.
- After the `dd` command ends, the Cache will no longer increase, but the block device write will continue for a while, and the result of multiple I/O writes will add up to the 500M data to be written by `dd`.

Comparing this result with the definition of Cache we just learned, you may be a
little dizzy. Why does the previous document say that Cache is a page cache for
file reading, but why is it also used for writing files now?

Let's write down this question for now, and then look at another case of disk writing.
After the two cases are over, we will conduct a unified analysis.

However, for the next case, I must emphasize one point:

The following commands have high requirements on the environment. You need to
configure multiple disks in your system, and the disk partition <ins>/dev/sdb1</ins>
must also be unused. Do not try this if you only have one disk, or you will
damage your disk partitions.

If your system meets the requirements, you can continue to run the following
command in the second terminal. After clearing the cache, write 2GB of random
data to the disk partition <ins>/dev/sdb1</ins>:

```shell
# 首先清理缓存
$ echo 3 > /proc/sys/vm/drop_caches
# 然后运行dd命令向磁盘分区/dev/sdb1写入2G数据
$ dd if=/dev/urandom of=/dev/sdb1 bs=1M count=2048
```

Then, go back to Terminal 1 and observe the changes in memory and I/O:

```shell
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 7584780 153592  97436    0    0   684     0   31  423  1 48 50  2  0
 1  0      0 7418580 315384 101668    0    0     0     0   32  144  0 50 50  0  0
 1  0      0 7253664 475844 106208    0    0     0     0   20  137  0 50 50  0  0
 1  0      0 7093352 631800 110520    0    0     0     0   23  223  0 50 50  0  0
 1  1      0 6930056 790520 114980    0    0     0 12804   23  168  0 50 42  9  0
 1  0      0 6757204 949240 119396    0    0     0 183804   24  191  0 53 26 21  0
 1  1      0 6591516 1107960 123840    0    0     0 77316   22  232  0 52 16 33  0
```

From here you will see that although the data is both written, the phenomena of
writing to disk and writing to files are still different. When writing to disk
(that is, when bo is greater than 0), both Buffer and Cache are growing, but
obviously Buffer grows much faster.

This shows that writing to disk uses a lot of Buffer, which is the same as the definition
we found in the document.

Comparing the two cases, we found that Cache is used to cache data when writing files,
while Buffer is used to cache data when writing to disk. So, back to the question
just now, although the document only mentions that Cache is a cache for file
reading, in fact, Cache also caches data when writing files.

### Scenario 2: Disk and file read case

After understanding the situation of disk and file writing, let's think about it
in reverse, what happens when disk and file are read?

We go back to the second terminal and run the command below. After clearing the
cache, read data from the file <ins>/tmp/file</ins> and write it to an empty device:

```shell
# 首先清理缓存
$ echo 3 > /proc/sys/vm/drop_caches
# 运行dd命令读取文件数据
$ dd if=/tmp/file of=/dev/null
```

Then, go back to Terminal 1 and observe the changes in memory and I/O:

```shell
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  1      0 7724164   2380 110844    0    0 16576     0   62  360  2  2 76 21  0
 0  1      0 7691544   2380 143472    0    0 32640     0   46  439  1  3 50 46  0
 0  1      0 7658736   2380 176204    0    0 32640     0   54  407  1  4 50 46  0
 0  1      0 7626052   2380 208908    0    0 32640    40   44  422  2  2 50 46  0
```

Observe the output of `vmstat`, you will find that when the file is read (that is,
when bi is greater than 0), the Buffer remains unchanged, while the Cache keeps
growing. This is consistent with the definition we found that "Cache is a page
cache for reading files".

So, what about disk reads? Let's run the second case to see.

First, go back to the second terminal and run the command below. After clearing
the cache, read data from the disk partition <ins>/dev/sda1</ins> and write to
the empty device:

```shell
# 首先清理缓存
$ echo 3 > /proc/sys/vm/drop_caches
# 运行dd命令读取磁盘
$ dd if=/dev/sda1 of=/dev/null bs=1M count=1024
```

Then, go back to Terminal 1 and observe the changes in memory and I/O:

```shell
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 7225880   2716 608184    0    0     0     0   48  159  0  0 100  0  0
 0  1      0 7199420  28644 608228    0    0 25928     0   60  252  0  1 65 35  0
 0  1      0 7167092  60900 608312    0    0 32256     0   54  269  0  1 50 49  0
 0  1      0 7134416  93572 608376    0    0 32672     0   53  253  0  0 51 49  0
 0  1      0 7101484 126320 608480    0    0 32748     0   80  414  0  1 50 49  0
```

Observing the output of `vmstat`, you will find that when reading the disk (that
is, when bi is greater than 0), both Buffer and Cache are growing, but obviously
the growth of Buffer is much faster. This means that when reading the disk, the
data is cached in the Buffer.

Of course, I think that after the analysis of the two cases in the previous scenario,
you can draw this conclusion by yourself: when reading a file, the data will be
cached in the Cache, and when reading the disk, the data will be cached in the Buffer.

At this point you should have found that although the documentation provides
descriptions of Buffer and Cache, it still cannot cover all the details. For
example, here are two things we learned today:

- Buffer can be used as both a "cache for data to be written to disk" and a "cache for reading data from disk".
- Cache can be used as both a "page cache for reading data from files" and a "page cache for writing files".

In this way, we have answered the two questions before the case begins.

Simply put, **Buffer is a cache for disk data, and Cache is a cache for file data.
They are used in both read requests and write requests**.

## Summary

Today, we explored the detailed meaning of Buffer and Cache in memory performance.
Buffer and Cache respectively cache the read and write data of disk and file system.

- From a writing point of view, it not only optimizes the writing of disks and files, but also benefits the application. The application can return to do other work before the data is actually placed on the disk.
- From the perspective of reading, it can not only speed up the reading of data that needs to be accessed frequently, but also reduce the pressure of frequent I/O on the disk.

In addition to the content of exploration itself, this exploration process should
also inspire you. When troubleshooting performance problems, because there are
too many performance indicators of various resources, it is impossible for us to
remember the detailed meanings of all indicators. Then, the accurate and efficient
method-checking documents is very important.

You must develop the habit of checking documents and learn to interpret the
detailed meaning of these performance indicators. In addition, the proc file system
is also a good helper for us. It shows us the internal running status of the system,
and is also the data source of many performance tools, which is a good way to
assist in troubleshooting performance problems.