# 02 Basics: How should we understand "average load"?

Whenever the system is slow, you probably execute the `top` or `uptime` command 
the first time.

Below is an example of the `uptime` command:

```shell
$ uptime
02:34:03 up 2 days, 20:14,  1 user,  load average: 0.63, 0.83, 0.88
```

The important thing is the last 3 numbers.
These 3 numbers are the average load (Load Average) of the past 1 minute, 5 
minutes, and 15 minutes in turn.

**Average load**? What's the meaning?

To put it simply, the average load refers to the average number of processes 
that the system is in a **runnable** and **uninterruptible** state per unit time,
that is, the **average number of active processes**, which is not directly related
to the CPU usage. Here I will first explain the words "runnable state" and
"uninterruptible state".

The so-called runnable process refers to the process that is using the CPU or
waiting for the CPU, that is, the process in the R state (Running or Runnable)
that we often see with the `ps` command.

The process in the uninterruptible state is the process in the key process of
the kernel state, and these processes are uninterruptible.
For example, the most common is to wait for the I/O response of the hardware
device, which is the process of D state (Uninterruptible Sleep, also known as
Disk Sleep ) in what we see in the `ps` command.

For example, when a process reads and writes data to the disk, in order to ensure
the consistency of the data, it cannot be interrupted by other processes or
interrupts before getting a reply from the disk.
At this time, the process is in an uninterruptible state. If the process at this
time is interrupted, it is prone to the problem of inconsistency between disk
data and process data.

Therefore, **the uninterruptible state is actually a protection mechanism of the
system for processes and hardware devices**.

Therefore, you can simply understand that the average load is actually the average
number of active processes.
The average number of active processes is intuitively understood as the number
of active processes per unit time, but it is actually the exponential decay
average of the number of active processes.
You don't have to worry about the detailed meaning of this "exponential decay average".
This is just a faster calculation method of the system, and it's okay if you
directly use it as the average of the number of active processes.

Since the average is the number of active processes, ideally, there is exactly
one process running on each CPU, so that each CPU is fully utilized.
For example, what does it mean when the load average is 2?

- On a system with only 2 CPUs, this means that all CPUs are exactly fully utilized.
- On a 4 CPU system, that means 50% of the CPU is free.
- In a system with only 1 CPU, it means that half of the processes cannot compete for the CPU.

## What is the reasonable load average

We know that the ideal load average is equal to the number of CPUs.
So when judging the average load, **you first need to know how many CPUs the
system has**, which can be read through the top command or from the file
<ins>/proc/cpuinfo</ins>, for example:

```shell
# For the usage of `grep` and `wc`,
# please consult their manuals or search the Internet
$ grep 'model name' /proc/cpuinfo | wc -l
2
```

With the number of CPUs, we can judge that when the average load is greater than
the number of CPUs, the system has already been overloaded.

However, wait a minute, a new problem came again.
We can see in the example that the average load has three values, which one
should we refer to?

In fact, it all depends. The average value of three different time intervals
actually provides us with a data source for **analyzing system load trends**,
allowing us to understand the current load status in a more comprehensive and
three-dimensional manner.

For example, just like the weather in Beijing in early autumn, if you only look
at the temperature at noon, you might think it is still summer in July.
But if you combine the temperature at three points in the morning, noon, and
night, you can basically understand the weather conditions of the day in all
directions.

The same is true for the three load time periods of the CPU mentioned above.

- If the three values of 1 minute, 5 minutes, and 15 minutes are basically the same, or the difference is not large, it means that the system load is very stable.
- But if the value of 1 minute is much smaller than the value of 15 minutes, it means that the load of the system has been decreasing in the last 1 minute, but there was a heavy load in the past 15 minutes.
- Conversely, if the value of 1 minute is much greater than the value of 15 minutes, it means that the load in the last 1 minute is increasing. This increase may be only temporary, or it may continue to increase, so continuous observation is required. Once the 1-minute average load approaches or exceeds the number of CPUs, it means that the system is overloaded. At this time, it is necessary to analyze and investigate where the problem is caused, and find a way to optimize it.

Let me give another example here, assuming we see load averages of 1.73, 0.60,
and 7.98 on a single-CPU system, it means that in the past 1 minute, the system
has 73% overload, and in 15 minutes, there is 698% overload.
From the perspective of the overall trend, the load of the system is decreasing.

So, in the actual production environment, when the average load is high, we need to focus on it?

In my opinion, **when the average load is higher than 70% of the number of CPUs**,
you should analyze and troubleshoot the problem of high load. Once the load is
too high, it may cause the process to respond slowly, thereby affecting the
normal function of the service.

But the figure of 70% is not absolute. The most recommended method is to monitor
the average load of the system, and then judge the change trend of the load based
on more historical data. When you find that the load has an obvious upward trend,
for example, the load has doubled, you can do analysis and investigation again.


### Load Average and CPU Utilization

In real work, we often confuse average load with CPU usage, so here, I also make
a distinction. You may be wondering, since the average load represents the
number of active processes, does a high average load mean high CPU usage?

We still have to go back to the meaning of load average, which refers to the
number of processes in a runnable and uninterruptible state per unit of time.
So, it includes not only the **processes that are using the CPU**, but also the
processes **waiting for the CPU** and **waiting for I/O**.

The CPU usage is the statistics of CPU busyness per unit time, which does not necessarily correspond exactly to the average load. for example:

- For a CPU-intensive process, using a large amount of CPU will lead to an increase in the average load, and the two are consistent at this time;
- For an I/O-intensive process, waiting for I/O will also lead to an increase in the average load, but the CPU usage is not necessarily very high.
- A large number of process scheduling waiting for the CPU will also lead to an increase in the average load, and the CPU usage will be relatively high at this time.

### Load Average Case Study

Below, we use three examples to look at these three situations, and use tools
such as `iostat`, `mpstat`, and `pidstat` to find out the root cause of the
increase in the average load.

Because the case analysis is based on the operation on the machine, it is not
enough to just listen and watch, it is better to follow me to actually operate it.

#### your preparation

The following cases are based on WSL2 Ubuntu 20.04, of course, it is also applicable
to other Linux systems. The case environment I am using is shown below.

- Machine configuration: 12 CPU, 32GB memory.
- Pre-install the `stress` and `sysstat` packages, e.g. `apt install stress sysstat`.

Here, I will briefly introduce `stress` and `sysstat`.

`stress` is a Linux system stress testing tool, here we use it as an abnormal
process to simulate the scenario where the average load increases.
And `sysstat` contains commonly used Linux performance tools to monitor and
analyze system performance. Our case will use the two commands `mpstat` and
`pidstat` of this package.

- `mpstat` is a commonly used multi-core CPU performance analysis tool, which is used to view the performance indicators of each CPU in real time, as well as the average indicators of all CPUs.
- `pidstat` is a commonly used process performance analysis tool, which is used to view the performance indicators of the process such as CPU, memory, I/O and context switching in real time.

In addition, each scenario requires you to open three terminals and log in to the same Linux machine.
Before the experiment, you should make the above preparations. If there is a problem
with the installation of the package, you can solve it yourself on Google first.
If you still can’t solve it, then come to the message area to find me. It shouldn’t be difficult.

Also note that all the following commands are run as the root user by default.
Therefore, if you log in to the system as an ordinary user, you must first run
the `sudo su root` command to switch to the root user.

If the above requirements have been fulfilled, you can first use the `uptime`
command to see the load average before the test:

```shell
uptime
...,  load average: 0.00, 0.00, 0.00
```

#### Scenario 1: CPU-intensive process

First, we run the stress command in the first terminal to simulate a 100% CPU usage scenario:

```shell
stress --cpu 1 --timeout 600
```

Next, run `uptime` in a second terminal to see how the load average changes:

```shell
# -d 参数表示高亮显示变化的区域
watch -d uptime
...,  load average: 0.76, 0.25, 0.09
```

Finally, run `mpstat` in a third terminal to see how the CPU usage changes:

```shell
# -P ALL 表示监控所有CPU，后面数字5表示间隔5秒后输出一组数据
mpstat -P ALL 5
Linux 5.15.90.1-microsoft-standard-WSL2 (MSI-TRIDENT3)  03/04/23        _x86_64_        (12 CPU)

23:28:55     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
23:29:00     all    8.27    0.00    0.15    0.02    0.00    0.70    0.00    0.00    0.00   90.87
23:29:00       0    0.00    0.00    0.00    0.00    0.00    7.78    0.00    0.00    0.00   92.22
23:29:00       1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
23:29:00       2    0.00    0.00    0.20    0.00    0.00    0.00    0.00    0.00    0.00   99.80
23:29:00       3    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
23:29:00       4  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
23:29:00       5    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
23:29:00       6    0.00    0.00    0.40    0.00    0.00    0.00    0.00    0.00    0.00   99.60
23:29:00       7    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
23:29:00       8    0.00    0.00    0.40    0.00    0.00    0.00    0.00    0.00    0.00   99.60
23:29:00       9    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
23:29:00      10    0.00    0.00    0.80    0.00    0.00    0.00    0.00    0.00    0.00   99.20
23:29:00      11    0.00    0.00    0.00    0.20    0.00    0.00    0.00    0.00    0.00   99.80
```

From Terminal 2, we can see that the 1-minute load average will slowly increase
to 1.00, and from Terminal 3, we can also see that there is exactly one CPU with
100% usage, but its iowait is only 0. This shows that the increase in load average
is due to 100% CPU usage.

So, which process is causing the 100% CPU usage? You can use pidstat to find out:

```shell
# 间隔5秒后输出一组数据
pidstat -u 5 1
Linux 5.15.90.1-microsoft-standard-WSL2 (MSI-TRIDENT3)  03/04/23        _x86_64_        (12 CPU)

23:32:21      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
23:32:26     1000      1855    0.00    0.20    0.00    0.00    0.20     2  wslconnect
23:32:26        0      3995  100.00    0.00    0.00    0.00  100.00     4  stress
23:32:26     1000      5784    0.00    0.20    0.00    0.00    0.20     6  bash

Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:     1000      1855    0.00    0.20    0.00    0.00    0.20     -  wslconnect
Average:        0      3995  100.00    0.00    0.00    0.00  100.00     -  stress
Average:     1000      5784    0.00    0.20    0.00    0.00    0.20     -  bash
```

From here it is obvious that the `stress` process is using 100% of the CPU.

#### Scenario 2: I/O intensive process

First run the `stress` command again, but this time to simulate I/O pressure,
that is, to execute **sync** continuously:

```shell
stress -i 1 --timeout 600
```

Still run `uptime` in the second terminal to see how the load average changes:

```shell
watch -d uptime
...,  load average: 0.91, 0.59, 0.26
```

Then, a third terminal runs `mpstat` to see how the CPU usage changes:

```shell
# 显示所有CPU的指标，并在间隔5秒输出一组数据
mpstat -P ALL 5 1
Linux 5.15.90.1-microsoft-standard-WSL2 (MSI-TRIDENT3)  03/08/23        _x86_64_        (12 CPU)

11:27:20     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
11:27:25     all    0.03    0.00    1.30    5.64    0.00    0.55    0.00    0.00    0.00   92.47
11:27:25       0    0.20    0.00    0.40    0.00    0.00    6.53    0.00    0.00    0.00   92.87
11:27:25       1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
11:27:25       2    0.00    0.00    0.20    0.00    0.00    0.00    0.00    0.00    0.00   99.80
11:27:25       3    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
11:27:25       4    0.00    0.00    0.20    0.00    0.00    0.00    0.00    0.00    0.00   99.80
11:27:25       5    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
11:27:25       6    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
11:27:25       7    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
11:27:25       8    0.00    0.00    4.09    0.00    0.00    0.00    0.00    0.00    0.00   95.91
11:27:25       9    0.00    0.00   10.36   67.33    0.00    0.00    0.00    0.00    0.00   22.31
11:27:25      10    0.20    0.00    0.40    0.00    0.00    0.00    0.00    0.00    0.00   99.40
11:27:25      11    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00

Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all    0.03    0.00    1.30    5.64    0.00    0.55    0.00    0.00    0.00   92.47
Average:       0    0.20    0.00    0.40    0.00    0.00    6.53    0.00    0.00    0.00   92.87
Average:       1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       2    0.00    0.00    0.20    0.00    0.00    0.00    0.00    0.00    0.00   99.80
Average:       3    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       4    0.00    0.00    0.20    0.00    0.00    0.00    0.00    0.00    0.00   99.80
Average:       5    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       6    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       7    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       8    0.00    0.00    4.09    0.00    0.00    0.00    0.00    0.00    0.00   95.91
Average:       9    0.00    0.00   10.36   67.33    0.00    0.00    0.00    0.00    0.00   22.31
Average:      10    0.20    0.00    0.40    0.00    0.00    0.00    0.00    0.00    0.00   99.40
Average:      11    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
```

It can be seen from here that the 1-minute average load will slowly increase to 1.06, and the system CPU usage of one of the CPUs rises to 23.87, while iowait reaches 67.53%. This shows that the increase in load average is due to the increase in iowait.

So which process is it that causes iowait to be so high? We still use `pidstat` to query:

```shell
# 间隔5秒后输出一组数据，-u表示CPU指标
pidstat -u 5 1
Linux 5.15.90.1-microsoft-standard-WSL2 (MSI-TRIDENT3)  03/08/23        _x86_64_        (12 CPU)

11:31:08      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
11:31:13     1000     17052    0.00    0.20    0.00    0.00    0.20    10  wslconnect
11:31:13        0     19972    0.00    6.80    0.00    0.00    6.80     1  stress
11:31:13     1000     23045    0.00    0.20    0.00    0.00    0.20     2  bash

Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:     1000     17052    0.00    0.20    0.00    0.00    0.20     -  wslconnect
Average:        0     19972    0.00    6.80    0.00    0.00    6.80     -  stress
Average:     1000     23045    0.00    0.20    0.00    0.00    0.20     -  bash
```

It can be found that it is caused by the stress process.

#### Scenario 3: Scenario with a large number of processes

A process waiting for the CPU occurs when there are more processes running on
the system than the CPU can run.

For example, we still use stress, but this time simulating 48 processes:

```shell
stress -c 48 --timeout 600
```

Since the system has only 12 CPUs, significantly less than 48 processes, the
CPUs of the system are severely overloaded with an average load of 42.90:

```shell
uptime
...,  load average: 42.90, 17.07, 6.32
```

Then run `pidstat` to see the status of the process:

```shell
# 间隔5秒后输出一组数据
pidstat -u 5 1
Linux 5.15.90.1-microsoft-standard-WSL2 (MSI-TRIDENT3)  03/08/23        _x86_64_        (12 CPU)

13:25:36      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
13:25:41        0      9790   24.80    0.00    0.00   75.80   24.80     0  stress
13:25:41        0      9791   25.00    0.00    0.00   74.80   25.00     9  stress
13:25:41        0      9792   25.00    0.20    0.00   75.20   25.20     7  stress
13:25:41        0      9793   25.00    0.00    0.00   75.20   25.00     4  stress
13:25:41        0      9794   25.00    0.00    0.00   74.80   25.00     4  stress
13:25:41        0      9795   25.00    0.00    0.00   74.80   25.00     8  stress
13:25:41        0      9796   25.00    0.00    0.00   75.00   25.00    11  stress
13:25:41        0      9797   25.00    0.00    0.00   75.40   25.00     2  stress
13:25:41        0      9798   24.40    0.20    0.00   75.00   24.60     0  stress
13:25:41        0      9799   25.00    0.00    0.00   74.80   25.00     8  stress
13:25:41        0      9800   25.00    0.00    0.00   74.80   25.00     7  stress
13:25:41        0      9801   25.00    0.00    0.00   75.00   25.00     2  stress
13:25:41        0      9802   24.60    0.00    0.00   75.80   24.60     1  stress
13:25:41        0      9803   25.00    0.00    0.00   75.40   25.00     9  stress
13:25:41        0      9804   25.00    0.00    0.00   75.00   25.00     4  stress
13:25:41        0      9805   25.00    0.00    0.00   75.20   25.00     3  stress
13:25:41        0      9806   24.80    0.00    0.00   75.20   24.80     7  stress
13:25:41        0      9807   25.00    0.00    0.00   74.60   25.00    11  stress
13:25:41        0      9808   25.00    0.00    0.00   75.20   25.00     8  stress
13:25:41        0      9809   25.00    0.00    0.00   75.20   25.00     6  stress
13:25:41        0      9810   25.00    0.00    0.00   74.40   25.00     1  stress
13:25:41        0      9811   25.00    0.00    0.00   75.00   25.00     4  stress
13:25:41        0      9812   25.00    0.00    0.00   75.20   25.00    11  stress
13:25:41        0      9813   25.00    0.00    0.00   75.00   25.00     3  stress
13:25:41        0      9814   25.00    0.00    0.00   75.40   25.00     2  stress
13:25:41        0      9815   24.60    0.20    0.00   76.40   24.80     0  stress
13:25:41        0      9816   24.60    0.00    0.00   75.60   24.60     1  stress
13:25:41        0      9817   25.00    0.00    0.00   75.20   25.00     9  stress
13:25:41        0      9818   25.00    0.00    0.00   75.00   25.00     5  stress
13:25:41        0      9819   24.80    0.20    0.00   75.00   25.00     8  stress
13:25:41        0      9820   25.00    0.00    0.00   75.00   25.00    10  stress
13:25:41        0      9821   25.00    0.00    0.00   74.60   25.00    10  stress
13:25:41        0      9822   24.80    0.20    0.00   75.40   25.00    11  stress
13:25:41        0      9823   25.00    0.00    0.00   75.40   25.00     6  stress
13:25:41        0      9824   25.00    0.00    0.00   75.00   25.00     2  stress
13:25:41        0      9825   25.20    0.00    0.00   74.80   25.20     3  stress
13:25:41        0      9826   25.00    0.00    0.00   74.60   25.00     9  stress
13:25:41        0      9827   25.20    0.00    0.00   74.80   25.20     6  stress
13:25:41        0      9828   25.20    0.00    0.00   74.80   25.20     5  stress
13:25:41        0      9829   25.00    0.00    0.00   75.20   25.00     3  stress
13:25:41        0      9830   25.00    0.00    0.00   75.40   25.00     1  stress
13:25:41        0      9831   24.80    0.00    0.00   75.40   24.80    10  stress
13:25:41        0      9832   25.00    0.00    0.00   75.40   25.00     5  stress
13:25:41        0      9833   25.00    0.00    0.00   75.00   25.00     7  stress
13:25:41        0      9834   24.80    0.00    0.00   74.80   24.80     0  stress
13:25:41        0      9835   25.00    0.00    0.00   75.20   25.00    10  stress
13:25:41        0      9836   25.00    0.00    0.00   74.80   25.00     6  stress
13:25:41        0      9837   25.00    0.00    0.00   75.00   25.00     5  stress
13:25:41     1000     17527    0.00    0.20    0.00    0.00    0.20     1  wslconnect

Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:        0      9790   24.80    0.00    0.00   75.80   24.80     -  stress
Average:        0      9791   25.00    0.00    0.00   74.80   25.00     -  stress
Average:        0      9792   25.00    0.20    0.00   75.20   25.20     -  stress
Average:        0      9793   25.00    0.00    0.00   75.20   25.00     -  stress
Average:        0      9794   25.00    0.00    0.00   74.80   25.00     -  stress
Average:        0      9795   25.00    0.00    0.00   74.80   25.00     -  stress
Average:        0      9796   25.00    0.00    0.00   75.00   25.00     -  stress
Average:        0      9797   25.00    0.00    0.00   75.40   25.00     -  stress
Average:        0      9798   24.40    0.20    0.00   75.00   24.60     -  stress
Average:        0      9799   25.00    0.00    0.00   74.80   25.00     -  stress
Average:        0      9800   25.00    0.00    0.00   74.80   25.00     -  stress
Average:        0      9801   25.00    0.00    0.00   75.00   25.00     -  stress
Average:        0      9802   24.60    0.00    0.00   75.80   24.60     -  stress
Average:        0      9803   25.00    0.00    0.00   75.40   25.00     -  stress
Average:        0      9804   25.00    0.00    0.00   75.00   25.00     -  stress
Average:        0      9805   25.00    0.00    0.00   75.20   25.00     -  stress
Average:        0      9806   24.80    0.00    0.00   75.20   24.80     -  stress
Average:        0      9807   25.00    0.00    0.00   74.60   25.00     -  stress
Average:        0      9808   25.00    0.00    0.00   75.20   25.00     -  stress
Average:        0      9809   25.00    0.00    0.00   75.20   25.00     -  stress
Average:        0      9810   25.00    0.00    0.00   74.40   25.00     -  stress
Average:        0      9811   25.00    0.00    0.00   75.00   25.00     -  stress
Average:        0      9812   25.00    0.00    0.00   75.20   25.00     -  stress
Average:        0      9813   25.00    0.00    0.00   75.00   25.00     -  stress
Average:        0      9814   25.00    0.00    0.00   75.40   25.00     -  stress
Average:        0      9815   24.60    0.20    0.00   76.40   24.80     -  stress
Average:        0      9816   24.60    0.00    0.00   75.60   24.60     -  stress
Average:        0      9817   25.00    0.00    0.00   75.20   25.00     -  stress
Average:        0      9818   25.00    0.00    0.00   75.00   25.00     -  stress
Average:        0      9819   24.80    0.20    0.00   75.00   25.00     -  stress
Average:        0      9820   25.00    0.00    0.00   75.00   25.00     -  stress
Average:        0      9821   25.00    0.00    0.00   74.60   25.00     -  stress
Average:        0      9822   24.80    0.20    0.00   75.40   25.00     -  stress
Average:        0      9823   25.00    0.00    0.00   75.40   25.00     -  stress
Average:        0      9824   25.00    0.00    0.00   75.00   25.00     -  stress
Average:        0      9825   25.20    0.00    0.00   74.80   25.20     -  stress
Average:        0      9826   25.00    0.00    0.00   74.60   25.00     -  stress
Average:        0      9827   25.20    0.00    0.00   74.80   25.20     -  stress
Average:        0      9828   25.20    0.00    0.00   74.80   25.20     -  stress
Average:        0      9829   25.00    0.00    0.00   75.20   25.00     -  stress
Average:        0      9830   25.00    0.00    0.00   75.40   25.00     -  stress
Average:        0      9831   24.80    0.00    0.00   75.40   24.80     -  stress
Average:        0      9832   25.00    0.00    0.00   75.40   25.00     -  stress
Average:        0      9833   25.00    0.00    0.00   75.00   25.00     -  stress
Average:        0      9834   24.80    0.00    0.00   74.80   24.80     -  stress
Average:        0      9835   25.00    0.00    0.00   75.20   25.00     -  stress
Average:        0      9836   25.00    0.00    0.00   74.80   25.00     -  stress
Average:        0      9837   25.00    0.00    0.00   75.00   25.00     -  stress
Average:     1000     17527    0.00    0.20    0.00    0.00    0.20     -  wslconnect
```

It can be seen that 48 processes are competing for 12 CPUs, and the time each
process waits for the CPU (that is, the %wait column in the code block) is as high as 75%.
These processes exceed the computing power of the CPU, eventually causing the
CPU to become overloaded.

## Summary

After analyzing these three cases, let me summarize the
**understanding of load average**.

The load average provides a quick view of the overall performance of the system,
reflecting the overall load situation. But just looking at the load average itself,
we can't directly find out where the bottleneck is. Therefore, when understanding
the average load, also pay attention to:

- A high load average may be caused by a CPU-intensive process;
- A high load average does not necessarily mean high CPU usage, it may also mean that I/O is busier;
- When you find that the load is high, you can use tools such as `mpstat` and `pidstat` to assist in analyzing the source of the load.