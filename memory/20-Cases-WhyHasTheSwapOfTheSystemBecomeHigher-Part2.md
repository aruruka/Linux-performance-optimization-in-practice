# 20 Cases: Why has the Swap of the system become higher? (Part 2)

In the previous section, we studied Linux memory reclamation in detail, especially
the principles of Swap, so let's start with a brief review.

When memory resources are tight, Linux frees up file pages and anonymous pages
through direct memory reclamation and periodic scans to allocate memory to
processes that need it more.

- The recycling of file pages is relatively easy to understand, just clear the cache directly, or write the dirty data back to disk and then release the cache.
- For anonymous pages that are not frequently accessed, they need to be swapped out to disk via Swap so that they can be swapped in from disk to memory again on the next access.

With Swap enabled, you can set <ins>/proc/sys/vm/min_free_kbytes</ins> to adjust
the threshold for the system to periodically reclaim memory, and <ins>/proc/sys/vm/swappiness</ins>
to adjust the tendency to reclaim file pages and anonymous pages.

So, how do you locate and analyze when Swap usage is elevated? Below, we'll look
at a case study of disk I/O for a hands-on analysis and walkthrough.

## Case

The following example is based on Ubuntu 18.04 and is applicable to other Linux systems as well.

- Machine configuration: 2 CPU, 8GB memory
- You need to pre-install a tool such as sysstat, e.g. `apt install sysstat`

First, we open two terminals, SSH to each machine, and install these tools
mentioned above.

As in the previous cases, all the next commands will be run as root by default.
If you are logged in as a normal user, please run sudo su root to switch to root.

If there are any problems in the installation process, again, you are encouraged
to search for solutions yourself first, and if you can't solve them, you can ask
me questions in the comments section.

Then, run the `free` command in the terminal to see how Swap is being used. For
example, on my machine, the output is as follows:

```shell
$ free
             total        used        free      shared  buff/cache   available
Mem:        8169348      331668     6715972         696     1121708     7522896
Swap:             0           0           0
```

As you can see from this free output, the Swap size is 0, which means that my
machine is not configured for Swap.

If you already have Swap enabled in your environment, you can skip the following
steps and move on.

To enable Swap, we first need to be clear that Linux itself supports two types
of Swap, namely Swap partitions and Swap files. Take the Swap file as an example,
run the following command in the first terminal to enable Swap, where I have
configured the Swap file to be 8GB in size:

```shell
# 创建Swap文件
$ fallocate -l 8G /mnt/swapfile
# 修改权限只有根用户可以访问
$ chmod 600 /mnt/swapfile
# 配置Swap文件
$ mkswap /mnt/swapfile
# 开启Swap
$ swapon /mnt/swapfile
```

Then, execute the `free` command again to confirm that the Swap configuration was
successful:

```shell
$ free
             total        used        free      shared  buff/cache   available
Mem:        8169348      331668     6715972         696     1121708     7522896
Swap:       8388604           0     8388604
```

Now, in the free output, Swap space and remaining space have changed from 0 to 8GB,
**indicating that Swap is properly enabled**.

Next, we run the following `dd` command in the first terminal to simulate the reading
of a large file:

```shell
# 写入空设备，实际上只有磁盘的读请求
$ dd if=/dev/sda1 of=/dev/null bs=1G count=2048
```

Next, run the `sar` command in a second terminal to see how the various memory
metrics are changing. You can watch a little longer to see how these metrics change.

```shell
# 间隔1秒输出一组数据
# -r表示显示内存使用情况，-S表示显示Swap使用情况
$ sar -r -S 1
04:39:56    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
04:39:57      6249676   6839824   1919632     23.50    740512     67316   1691736     10.22    815156    841868         4

04:39:56    kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
04:39:57      8388604         0      0.00         0      0.00

04:39:57    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
04:39:58      6184472   6807064   1984836     24.30    772768     67380   1691736     10.22    847932    874224        20

04:39:57    kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
04:39:58      8388604         0      0.00         0      0.00

…


04:44:06    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
04:44:07       152780   6525716   8016528     98.13   6530440     51316   1691736     10.22    867124   6869332         0

04:44:06    kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
04:44:07      8384508      4096      0.05        52      1.27
```

We can see that the output of `sar` is in two tables, the first table represents
the memory usage and the second table represents the Swap usage. The kb prefix
in front of each metric name indicates that the units of these metrics are KB.

After removing the prefix, you will find that most of the indicators we have already
seen, leaving a few emerging indicators that I will briefly introduce.

- kbcommit, which indicates how much memory is needed for the current system load. It is actually an estimate of how much memory is needed to ensure that the system memory does not overflow. %commit, is the percentage of this value relative to the total memory.
- kbactive, indicates active memory, that is, memory that has been used recently and normally will not be reclaimed by the system.
- kbinact, indicates inactive memory, that is, memory that is not accessed frequently and may be reclaimed by the system.

Once the meaning of the interface metrics is clear, let's analyze the phenomenon
in relation to the specific values. You can clearly see that the total memory usage
(%memused) is growing, from 23% at the beginning all the way up to 98%, and that
the main memory is occupied by buffers (kbbuffers). Specifically:

- At the beginning, the remaining memory (kbmemfree) keeps decreasing while the buffers (kbbuffers) keep increasing, so it is clear that the remaining memory is constantly allocated to buffers.
- After a while, the remaining memory is already very small and the buffers take up most of the memory. At this point, Swap usage starts to gradually increase, while the buffer and remaining memory fluctuate in a small range.

You may be confused as to why the buffer keeps growing. And which processes are
causing this?

Obviously, we also have to look at the process cache. As we learned in the caching
example earlier, `cachetop` does exactly that. So let's take a look at `cachetop`.

In the second terminal, press Ctrl+C to stop the `sar` command, and then run the
following `cachetop` command to observe the cache usage:

```shell
$ cachetop 5
12:28:28 Buffers MB: 6349 / Cached MB: 87 / Sort: HITS / Order: ascending
PID      UID      CMD              HITS     MISSES   DIRTIES  READ_HIT%  WRITE_HIT%
   18280 root     python                 22        0        0     100.0%       0.0%
   18279 root     dd                  41088    41022        0      50.0%      50.0%
```

Using the output of `cachetop`, we see that the `dd` process has only a 50% hit
rate for read and write requests, and the number of cache pages not hit (MISSES)
is 41022 (in pages). This suggests that it is the `dd` running at the beginning
of the case that is causing the elevated buffer usage.

You may then ask, "Why is the swap elevated as well? Intuitively, buffers take up
most of the system's memory and are also recyclable memory, so when memory runs out,
shouldn't buffers be recycled first?

In this case, we have to further look at the remaining memory, memory thresholds,
and the activity of anonymous and file pages via <ins>/proc/zoneinfo</ins>.

You can stop the `cachetop` command by pressing Ctrl+C in the second terminal.
Then run the following command to see how these indicators change in <ins>/proc/zoneinfo</ins>:

```shell
# -d 表示高亮变化的字段
# -A 表示仅显示Normal行以及之后的15行输出
$ watch -d grep -A 15 'Normal' /proc/zoneinfo
Node 0, zone   Normal
  pages free     21328
        min      14896
        low      18620
        high     22344
        spanned  1835008
        present  1835008
        managed  1796710
        protection: (0, 0, 0, 0, 0)
      nr_free_pages 21328
      nr_zone_inactive_anon 79776
      nr_zone_active_anon 206854
      nr_zone_inactive_file 918561
      nr_zone_active_file 496695
      nr_zone_unevictable 2251
      nr_zone_write_pending 0
```

You can notice that the remaining memory (pages_free) fluctuates constantly
within a small range. When it is less than the pages_low threshold, it suddenly
increases to a value greater than the pages_high threshold.

Combining this with the changes in remaining memory and buffers we just saw with
`sar`, we can deduce that the fluctuating changes in remaining memory and buffers
are due to the very cycle of memory reclaiming and cache reallocation.

- When the remaining memory is less than the page-low threshold, the system reclaims some cache and anonymous memory, making the remaining memory larger. In this case, the cache recycling leads to a reduction of the buffer in `sar`, while the anonymous memory recycling leads to an increase of the Swap usage.
- Immediately afterwards, as `dd` continues, the remaining memory is reallocated to the cache, causing the remaining memory to decrease and the buffer to increase.

In fact, there is another interesting phenomenon. If you run `dd` and `sar` many
times, you may find that sometimes Swap is used more, sometimes Swap is used less,
and the buffer fluctuates more during the many loop repetitions.

In other words, when the system recovers memory, sometimes it recovers more file pages,
and sometimes it recovers more anonymous pages.

Apparently, the tendency of the system to recycle different types of memory seems
to be less obvious. You should have thought of the swappiness mentioned in the
previous lesson, which is exactly the configuration option for adjusting different
types of memory recycling.

Still in the second terminal, press Ctrl+C to stop the watch command, and then
run the following command to view the swappiness configuration:

```shell
$ cat /proc/sys/vm/swappiness
60
```

swappiness shows the default value of 60, which is a relatively neutral configuration,
so the system will choose the appropriate type of recycling, such as recycling
inactive anonymous pages, or inactive file pages, depending on the actual operation.

At this point, we have identified the root cause of the Swap. Another question is,
which applications were affected by the swap? In other words, which processes'
memory is being swapped out by the Swap?

Here I still recommend the proc file system to see the size of the virtual memory
swapped out by the process Swap, which is stored in VmSwap in <ins>/proc/pid/status</ins>
(it is recommended that you run `man proc` to look up the meaning of the other fields).

```shell
# 按VmSwap使用量对进程排序，输出进程名称、进程ID以及SWAP用量
$ for file in /proc/*/status ; do awk '/VmSwap|Name|^Pid/{printf $2 " " $3}END{ print ""}' $file; done | sort -k 3 -n -r | head
dockerd 2226 10728 kB
docker-containe 2251 8516 kB
snapd 936 4020 kB
networkd-dispat 911 836 kB
polkitd 1004 44 kB
```

As you can see here, it is the dockerd and docker-containe processes that use
Swap more, so when dockerd accesses the memory swapped out to disk again, it
will also be slower.

This also illustrates the point that although caches are recyclable memory, in
scenarios like large file copies, the system uses the Swap mechanism to reclaim
anonymous memory, not just the file pages that take up the majority of memory.

Finally, if you configured Swap at the beginning, don't forget to turn it off at
the end of the case. You can run the following command to close Swap:

```shell
$ swapoff -a
```

In fact, closing the Swap and reopening it is a common way to clean up the Swap
space, e.g:

```shell
$ swapoff -a && swapon -a 
```

## Summary

When memory resources are tight, Linux will swap out infrequently accessed anonymous
pages to disk and swap them into memory from disk the next time they are accessed.
You can set <ins>/proc/sys/vm/min_free_kbytes</ins> to adjust the threshold for
the system to reclaim memory periodically, and <ins>/proc/sys/vm/swappiness</ins>
to adjust the tendency to reclaim file pages and anonymous pages.

When Swap gets high, you can use `sar`, <ins>/proc/zoneinfo</ins>, <ins>/proc/pid/status</ins>,
etc. to see the memory usage of your system and processes to find out the root
cause of the Swap increase and the affected processes.

On the flip side, it is often the case that reducing the use of Swap improves the
overall performance of the system. How to do it? Here, I have also summarized a
few common ways to reduce it.

- Disabling Swap, nowadays, servers have enough memory so that disabling Swap is fine unless necessary. With the popularity of cloud computing, Swap is disabled by default for virtual machines in most cloud platforms.
- If you really need to use Swap, you can try to reduce the value of swappiness to reduce the tendency of Swap usage during memory reclamation.
- In response to latency-sensitive applications, if they may be running in a Swap-enabled server, you can also lock memory with the library function `mlock()` or `mlockall()` to prevent them from swapping out of memory.