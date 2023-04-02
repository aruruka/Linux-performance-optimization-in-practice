# 13 Q&A(1): What should I do if the problem of RES interruption cannot be simulated?

Since the column was updated, we have finished learning CPU performance, one of
the four basic modules. I am very happy that more than half of the students have
not fallen behind, are still learning, actively practicing operations, and left
a lot of enthusiastic messages.

In these comments, I am very happy to see that many students have already learned
and applied them flexibly. Using the case ideas they have learned, they have
analyzed the performance bottlenecks of online applications and solved performance
problems in actual work. There are also students who can think over and over
again and point out some inappropriate or imprecise narratives in the article.
I am also very grateful to you, and I am happy to discuss with you.

In addition, many questions raised in the comments are also very valuable. I have
already answered most of them in the app. reply. On the other hand, it is also
to ensure that no one misses any key points.

Today is the first issue of performance optimization Q&A. In order to facilitate
your study and understanding, they are not strictly arranged in the order of the
articles. For each question, I have attached a screenshot of the question in the
comment area. If you need to review the original content, you can scan the QR
code at the bottom right of each question to view it.

## Problem 1: The performance tool version is too low, resulting in incomplete indicators

This is a common problem encountered by students who use CentOS. In the article,
there is a %wait indicator in my `pidstat` output, which represents the percentage
of time the process waits for the CPU. This is a new indicator introduced in systat
version 11.5.5, and the old version does not have this item. The sysstat version
in the CentOS software library is just lower than this, so there is no such indicator.

You don't have to worry, though. As I have emphasized before, tools are just means
of search and analysis, and indicators are the object of our key analysis. If it
is not displayed in your `pidstat`, there are naturally other ways to find this indicator.

For example, when explaining system principles and performance tools, I will
generally introduce some knowledge of the **proc file system** and teach you to
understand the various indicators provided by the proc file system. The reason
for this, on the one hand, of course, is to let you understand the working principle
of the system more intuitively; on the other hand, it actually wants to show you
the original data source of various performance indicators that can be seen on
the performance tool.

In this way, in the actual production environment, even if you probably need to
run an old version of the operating system and do not have permission to install
new software packages, you can still check the proc file system to get the
indicators you want.

However, for the learning of performance analysis, I still recommend that you
use the latest performance tools to learn. The new tool has more comprehensive
indicators, making it easier for you to start analysis. This absolute advantage
allows you to get the data you want more intuitively, and it is not easy for you
to retreat.

Of course, when you are a beginner, you'd better try to understand the principles
of performance tools, or go back and re-learn the principles after you are
familiar with the usage methods. This way, even in an environment where new tools
cannot be installed, you can still get the same metrics from the proc file
system or elsewhere for effective analysis.

## Problem 2: Using the `stress` command, the scene with high iowait cannot be simulated

- > Measured stress -i 1 --timeout 600, although it prompts "dispatching hogs: O cpu, 1 io", through mpstat - P ALL, the feedback is still the high load of %sys, not %iowait???
- > Conduct experiment 2 stress -i 1 --timeout 600 to simulate sync, and the average load has indeed increased, but in mpstst -P ALL 5 1, check that the sys column is close to 100% instead of iowait.

You can't simulate iowait raising with `stress`, but you see sys raising. This is
because of the `stress -i` option in the case, which indicates that the problem
of simulating I/O through the system call `sync()`, but this method is actually
not reliable.

Because the original intention of `sync()` is to flush the data in the memory
buffer to the disk to ensure synchronization. If there is not much data in the
buffer, then there is not much data to read and write to the disk, and there is
no way to generate I/O pressure.

This is especially obvious in the environment where SSD disks are used. It is
likely that your iowait is always 0, but simply because of a large number of
system calls, the system CPU usage sys increases.

In this case, I also replied in the message, recommending to use `stress-ng`
instead of `stress`. I'm worried that you didn't see the message, so I'll
emphasize it again here.

You can run the following command to simulate the problem of iowait.

```shell
# -i的含义还是调用sync，而—hdd则表示读写临时文件
$ stress-ng -i 1 --hdd 1 --timeout 600
```

## Problem 3: Unable to simulate the problem of RES interruption

> Teacher, I use ubuntu 16.04 vps, single core, 2G memory, no matter whether the test program is running or not, RES is 0, in has no obvious change, fluctuating between 100~400, cs has surged from hundreds to hundreds of thousands. I would like to ask why there is no obvious interruption problem on my side? And my test program will stop very quickly. Is it because the vps configuration is too low?

The problem is that even if a large number of threads are running, it is impossible
to simulate the problem of rescheduling interrupt RES rising.

In fact, I have already mentioned in the case of CPU context switching that the
rescheduling interrupt is a mechanism used by the scheduler to disperse tasks to
different CPUs, that is, it can wake up the CPU in the idle state to schedule
new tasks to run, and this usually uses **interprocessor Interrupts
(Inter-Processor Interrupts, IPI)** to achieve.

Therefore, this interrupt is of course meaningless on a single-core (only one
logical CPU) machine, because there will be no rescheduling at all.

However, as the message said, the problem of context switching still exists, so
you will see that cs (context switch) has increased from a few hundred to hundreds
of thousands, and at the same time, the voluntary context switching and involuntary
context switching of `sysbench` threads will also increase significantly, especially
involuntary context switching, will rise to hundreds of thousands. We all know,
by the meaning of the involuntary context, that this is too many threads
competing for the CPU.

In fact, this conclusion can also be obtained from another angle. For example,
you can add -u and -t parameters to the options of `pidstat` to output the CPU
usage of threads, and you will see the following interface:

```shell
$ pidstat -u -t 1

14:24:03      UID      TGID       TID    %usr %system  %guest   %wait    %CPU   CPU  Command
14:24:04        0         -      2472    0.99    8.91    0.00   77.23    9.90     0  |__sysbench
14:24:04        0         -      2473    0.99    8.91    0.00   68.32    9.90     0  |__sysbench
14:24:04        0         -      2474    0.99    7.92    0.00   75.25    8.91     0  |__sysbench
14:24:04        0         -      2475    2.97    6.93    0.00   70.30    9.90     0  |__sysbench
14:24:04        0         -      2476    2.97    6.93    0.00   68.32    9.90     0  |__sysbench
...
```

From the output interface of this `pidstat`, you can find that the %wait of each
`stress` thread is as high as 70%, while the CPU usage is less than 10%. In other
words, the stress thread spends most of its time waiting for the CPU, which also
shows that there are indeed too many threads competing for the CPU.

Here, by the way, is a very common mistake in comments. Some students will compare
%wait in `pidstat` with iowait% (abbreviated as wa) in `top`. In fact, this is
meaningless, because they are two completely irrelevant indicators.

- In `pidstat`, %wait indicates the percentage of time the process is waiting for the CPU.
- In `top`, iowait% indicates the percentage of CPU time waiting for I/O.

Recalling the process state we have learned, you should remember that the process
waiting for the CPU is already in the ready queue of the CPU and is in the running
state; while the process waiting for I/O is in the uninterruptible state.

In addition, the operating parameters of different versions of `sysbench` are not
exactly the same. For example, in the case of Ubuntu 18.04, the format to run
`sysbench` is:

```shell
$ sysbench --threads=10 --max-time=300 threads run
```

In Ubuntu 16.04, the running format is (thanks to Haku for sharing the execution
command):

```shell
$ sysbench --num-threads=10 --max-time=300 --test=threads run
```

## Problem 4: Unable to simulate I/O performance bottlenecks and excessive I/O pressure

> The I/O bottleneck of the system has not been simulated, can you help me take a
> look: only one app process is found after the container is running

This question can be seen as an extension of the previous question, except that
the `stress` command is replaced by an app running in a container.

In fact, in the case of I/O bottlenecks, in addition to the above message that
the simulation was unsuccessful, there are more messages that are just the opposite,
saying that the I/O pressure of the case is too high, causing various problems
on your own machine. Not even the system is responding.

The reason for this is actually because everyone's machine configuration is different,
including the difference in CPU and memory configuration, but also because of the
huge difference in disk. For example, the performance difference between mechanical
disk (HDD), low-end solid-state disk (SSD) and high-end solid-state disk may
reach several times to dozens of times.

In fact, the case machine I use is only a low-end SSD, which is slightly better
than a mechanical disk, but it is still not comparable to a high-end solid-state disk.
Therefore, under the same operation, I just had an I/O bottleneck on my machine,
but if I switch to a machine that uses mechanical disks, the disk I/O may be
crushed to death (expressed as 100% usage for a long time), and If you replace
it with a better SSD disk, it may not be able to generate enough I/O pressure.