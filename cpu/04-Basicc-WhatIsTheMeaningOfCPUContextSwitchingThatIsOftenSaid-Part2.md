# 04 Basics: What is the meaning of CPU context switching that is often said? (Part 2)

In the previous section, I told you how CPU context switching works.
To briefly review, CPU context switching is a core function to ensure the normal
operation of the Linux system. According to different scenarios, it can be divided
into process context switching, thread context switching and interrupt context switching.
You also have to go through the specific concepts and differences in your mind.
If you forget, check the previous article in time.

Today we will continue to look at how to analyze the problem of CPU context switching.

## How to check the context switching of the system?

Through the previous study, we know that too much context switching will consume
CPU time on saving and restoring data such as registers, kernel stacks, and virtual memory,
shortening the actual running time of the process, and becoming a culprit for a
sharp drop in system performance.

Since context switching has such a great impact on system performance, you must
be eager to know how to check context switching? Here, we can use the tool `vmstat`
to query the context switching of the system.

`vmstat` is a commonly used system performance analysis tool, mainly used to
analyze the memory usage of the system, and also used to analyze the number of
CPU context switches and interrupts.

For example, the following is an example of using `vmstat`:

```shell
# 每隔5秒输出1组数据
sudo vmstat 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 25226624 353224 6784140    0    0    21    78   11   59  0  0 100  0  0
 0  0      0 25223716 353224 6784180    0    0     0     0  139  536  0  0 100  0  0
```

Let's look at the result together, you can first try to interpret the meaning of
each column yourself. Here, I will highlight the four columns that require special attention:

- cs (context switch) is the number of context switches per second.
- in (interrupt) is the number of interrupts per second.
- r (Running or Runnable) is the length of the ready queue, that is, the number of processes that are running and waiting for the CPU.
- b (Blocked) is the number of processes in an uninterruptible sleep state.

It can be seen that the number of context switches `cs` in this example is 59 times,
the number of system interrupts `in` is 11 times, and the length of the ready queue `r`
is 1, and the number of uninterruptible state processes `b` is 0.


`vmstat` only gives the overall context switching status of the system.
If you want to view the details of each process, you need to use the `pidstat`
we mentioned earlier. Add the `-w` option to it, and you can see the context switching of each process.

For example:

```shell
# 每隔5秒输出1组数据
$ pidstat -w 5
Linux 5.15.90.1-microsoft-standard-WSL2 (MSI-TRIDENT3)  03/13/23        _x86_64_        (12 CPU)

14:58:15      UID       PID   cswch/s nvcswch/s  Command
14:58:20     1000        10     10.00      0.00  wslconnect
14:58:20     1000        47      6.00      0.00  bash
14:58:20        0      2200      0.20      0.00  pidstat
14:58:20     1000      2235      0.20      0.00  sleep
...
Average:      UID       PID   cswch/s nvcswch/s  Command
Average:     1000        10      7.80      0.00  wslconnect
Average:     1000        47      5.60      0.07  bash
Average:        0      2200      0.20      0.00  pidstat
Average:     1000      2289      0.07      0.00  sleep
```

There are two columns in this result that we focus on. One is `cswch`, which indicates
the number of voluntary context switches per second, and the other is `nvcswch`,
which indicates the number of non-voluntary context switches per second.

You must keep these two concepts in mind, because they imply different performance issues:

- The so-called **voluntary context switching refers to the context switching caused by the inability of the process to obtain the required resources**. For example, voluntary context switching occurs when system resources such as I/O and memory are insufficient.
- **Non-voluntary context switching refers to the context switching that occurs because the process is forced to be scheduled by the system due to reasons such as the time slice has expired**. For example, when a large number of processes are competing for the CPU, involuntary context switches are prone to occur.

### Case analysis

Knowing how to view these indicators, another question arises, how often is the
context switching frequency considered normal?
Don't rush for an answer. Similarly, let's look at a case of context switching first.
Through case exercises, you can analyze and find out this standard by yourself.

#### Your preparation

In today's case, we will use `sysbench` to simulate the situation of system multi-thread scheduling switching.

`sysbench` is a multi-threaded benchmark tool, generally used to evaluate the database load
under different system parameters.
Of course, in this case, we only regard it as an abnormal process, and its function
is to simulate the problem of excessive context switching.

The following example is based on WSL2 Ubuntu 20.04, of course, other Linux systems are also applicable.
The case environment I'm using looks like this:

- Machine configuration: 2 CPU, 8GB memory.
- Preinstall the `sysbench` and `sysstat` packages, e.g. `apt install sysbench sysstat`

Before the real operation starts, you need to open three terminals, log in to the same Linux machine,
and install the two software packages mentioned above.
For the installation of the package, you can Google it first and solve it by yourself.
If you still have problems, write down your situation in the message area.

Also note that all the following commands are **run as the root user by default**.
Therefore, if you log in to the system as a normal user, remember to run
the `sudo su root` command to switch to the root user first.

After the installation is complete, you can first use `vmstat` to see the number
of context switches on the idle system:

```shell
# 间隔1秒后输出1组数据
$ vmstat 1 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 6984064  92668 830896    0    0     2    19   19   35  1  0 99  0  0
```

Here you can see that the current number of context switches `cs` is 35, while
the number of interrupts `in` is 19, `r` and `b` are both 0.
Since I'm not running any other tasks at the moment, they are the number of context switches from an idle system.

#### Operation and analysis

Next, we start enter the actual combat operation.
First, run `sysbench` in the first terminal to simulate the bottleneck of the system's multi-thread scheduling:

```shell
# 以10个线程运行5分钟的基准测试，模拟多线程切换的问题
$ sysbench --threads=10 --max-time=300 threads run
```

Next, run `vmstat` in a second terminal to watch for context switches:

```shell
# 每隔1秒输出1组数据（需要Ctrl+C才结束）
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 6  0      0 6487428 118240 1292772    0    0     0     0 9019 1398830 16 84  0  0  0
 8  0      0 6487428 118240 1292772    0    0     0     0 10191 1392312 16 84  0  0  0
```

You should notice that the number of context switches `in` the `cs` column has
jumped from 35 to 1.39 million. At the same time, pay attention to observe several other indicators:

- Column `r`: The length of the ready queue has reached 8, far exceeding the number of CPUs in the system 2, so there must be a lot of CPU competition.
- `us` (user) and `sy` (system) columns: the CPU usage of these two columns adds up to 100%, and the system CPU usage, that is, the `sy` column is as high as 84%, indicating that the CPU is mainly occupied by the kernel.
- `in` column: The number of interrupts has also risen to about 10,000, indicating that interrupt handling is also a potential problem.

Combining these indicators, we can know that the ready queue of the system is too long,
that is, the number of processes running and waiting for the CPU is too large,
resulting in a large number of context switches, which in turn lead to an increase
in the CPU usage of the system.

So what process is causing these problems?

Let's continue the analysis, and then use `pidstat` in the third terminal to see
the situation of CPU and process context switching:

```shell
# 每隔1秒输出1组数据（需要 Ctrl+C 才结束）
# -w参数表示输出进程切换指标，而-u参数则表示输出CPU使用指标
$ pidstat -w -u 1
08:06:33      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
08:06:34        0     10488   30.00  100.00    0.00    0.00  100.00     0  sysbench
08:06:34        0     26326    0.00    1.00    0.00    0.00    1.00     0  kworker/u4:2

08:06:33      UID       PID   cswch/s nvcswch/s  Command
08:06:34        0         8     11.00      0.00  rcu_sched
08:06:34        0        16      1.00      0.00  ksoftirqd/1
08:06:34        0       471      1.00      0.00  hv_balloon
08:06:34        0      1230      1.00      0.00  iscsid
08:06:34        0      4089      1.00      0.00  kworker/1:5
08:06:34        0      4333      1.00      0.00  kworker/0:3
08:06:34        0     10499      1.00    224.00  pidstat
08:06:34        0     26326    236.00      0.00  kworker/u4:2
08:06:34     1000     26784    223.00      0.00  sshd
```

From the output of `pidstat`, you can find that the increase in CPU usage is indeed
caused by `sysbench`, and its CPU usage has reached 100%. But context switches
come from other processes, including `pidstat` with the most involuntary context switches,
and the kernel threads `kworker` and `sshd` with the most voluntary context switches.

However, if you are careful, you must have discovered a weird thing: the number
of context switches output by `pidstat` adds up to only a few hundred, which is
significantly smaller than the 1.39 million of `vmstat`. What's going on here?
Could it be something wrong with the tool itself?

Don't worry, before doubting the tool, let's recall the several context switching
scenarios mentioned earlier. One point mentions that the basic unit of Linux
scheduling is actually a thread, and our scenario `sysbench` simulates the thread
scheduling problem, so, does `pidstat` ignore the thread data?

By running `man pidstat`, you will find that `pidstat` displays the indicator data
of the process by default, and the indicator of the thread will be output only
after the `-t` parameter is added.

Therefore, in the third terminal, we can stop the `pidstat` command just now with Ctrl+C,
add the `-t` parameter, and try again:

```shell
# 每隔1秒输出一组数据（需要 Ctrl+C 才结束）
# -wt 参数表示输出线程的上下文切换指标
$ pidstat -wt 1
08:14:05      UID      TGID       TID   cswch/s nvcswch/s  Command
...
08:14:05        0     10551         -      6.00      0.00  sysbench
08:14:05        0         -     10551      6.00      0.00  |__sysbench
08:14:05        0         -     10552  18911.00 103740.00  |__sysbench
08:14:05        0         -     10553  18915.00 100955.00  |__sysbench
08:14:05        0         -     10554  18827.00 103954.00  |__sysbench
...
```

Now you can see that although the `sysbench` process (that is, the main thread)
does not seem to have many context switches, its child threads have a lot of
context switches. It seems that the context switch culprit is too many `sysbench`
threads.

We have found the root cause of the increase in the number of context switches,
so can we end here?

of course not. I don’t know if you still remember, when observing the system
indicators earlier, in addition to the sudden increase in the frequency of context
switching, there is another indicator that has also changed a lot. Yes, exactly
the number of interruptions. The number of interupt also rose to 10,000, but
exactly what type of interupt went up is unclear. Let's continue to peel off the
cocoons to find the source.

Since it is an interrupt, we all know that it only occurs in the kernel mode,
and `pidstat` is only a process performance analysis tool, and does not provide
any detailed information about the interrupt. How can we know the type of interrupt?

That's right, it is read from the read-only file <ins>/proc/interrupts</ins>.
<ins>/proc</ins> is actually a virtual file system of Linux, used for communication
between kernel space and user space. <ins>/proc/interrupts</ins> is part of this
communication mechanism, providing a read-only interrupt usage.

We are still in the third terminal, Ctrl+C to stop the `pidstat` command just now,
and then run the following command to observe the change of the interruption:

```shell
# -d 参数表示高亮显示变化的区域
$ watch -d cat /proc/interrupts
           CPU0       CPU1
...
RES:    2450431    5279697   Rescheduling interrupts
...
```

After observing for a period of time, you can find that the fastest changing is
the **rescheduling interrupt** (RES). This type of interrupt means that the CPU
in the idle state is woken up to schedule a new task to run. This is the mechanism
used by the scheduler to disperse tasks to different CPUs in a multi-processor
system (SMP). It is also commonly referred to as **Inter-Processor Interrupts**
(IPI).

Therefore, the increase in interrupts here is still due to the scheduling
problem of too many tasks, which is consistent with the previous analysis results
of the number of context switches.

Through this case, you should have also discovered the benefits of comparing
observations with multiple tools and multiple indicators. If we only used `pidstat`
to observe at the beginning, these very serious context switching threads would
not be found at all.

Now back to the original question, how many context switches per second is normal?

**This value actually depends on the CPU performance of the system itself.**
In my opinion, if the number of context switches of the system is relatively stable,
it should be considered normal if it ranges from hundreds to less than 10,000.
But when the number of context switches exceeds 10,000, or the number of switches
increases by an order of magnitude, there is likely to be a performance problem.

At this time, you also need to make a specific analysis according to the type of
context switching. for example:

- There are more voluntary context switches, indicating that processes are waiting for resources, and other problems such as I/O may have occurred;
- There are more involuntary context switches, which means that processes are being forcibly scheduled, that is, they are all competing for the CPU, which means that the CPU has indeed become a bottleneck;
- If the number of interrupts increases, it means that the CPU is occupied by the interrupt handler. It is also necessary to analyze the specific interrupt type by viewing the /proc/interrupts file.

## Summary

Today, through a `sysbench` case, I have told you about the analysis of context
switching problems. When encountering the problem of too many context switches,
we can use tools such as `vmstat`, `pidstat` and <ins>/proc/interrupts</ins>
to assist in troubleshooting the root cause of the performance problem.
