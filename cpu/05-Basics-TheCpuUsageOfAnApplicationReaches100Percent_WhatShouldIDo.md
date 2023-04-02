# 05 Basics: The CPU usage of an application reaches 100%, what should I do?

Through the study of load average and CPU context switching in the first two sections,
I believe you have a preliminary understanding of CPU performance. But I still
want to ask, before studying this column, what indicator do you most often use
to describe the CPU performance of the system? I think your answer may not be
average load, nor CPU context switching, but another more intuitive indicator - CPU usage.

As we said earlier, CPU usage is the statistics of CPU usage per unit time, displayed in percentage.
So, as the most commonly used and familiar CPU indicator, can you tell how the
CPU usage is calculated?
Also, can you figure out the difference between %user, %nice, %system, %iowait,
%steal, etc. displayed by performance tools such as `top` and `ps`?

Today I will take you to understand the content of CPU usage. At the same time,
I will take our most commonly used reverse proxy server Nginx as an example to
give you a deep understanding in step-by-step operations and analysis.

## CPU usage

In the previous issue, I mentioned that Linux, as a multitasking operating system,
divides the time of each CPU into very short time slices, and then allocates
them to each task in turn through the scheduler, thus creating the illusion that
multiple tasks are running at the same time.

In order to maintain CPU time, Linux triggers a time interrupt through a pre-defined
tick rate (represented as HZ in the kernel), and uses the global variable Jiffies
to record the number of ticks since booting. Each time a interrupt occurs, the
value of Jiffies is incremented by 1.

Tick rate HZ is a configurable option of the kernel and can be set to 100, 250,
1000, etc. Different systems may set different values, you can check its configuration
value by querying the <ins>/boot/config</ins> kernel option. For example,
in my system, the tick rate is set to 250, that is, 250 time interrupts are triggered per second.

```shell
$ grep 'CONFIG_HZ=' /boot/config-$(uname -r)
CONFIG_HZ=250
```

At the same time, because the tick rate HZ is a kernel option, user space programs
cannot directly access it. For the convenience of user space programs, the kernel
also provides a user space tick rate USER_HZ, which is always fixed at 100,
which is 1/100 second. In this way, the user space program does not need to care
how much HZ is set in the kernel, because it always sees the fixed value USER_HZ.

Linux provides information about the internal state of the system to user space
through the <ins>/proc</ins> virtual file system, and <ins>/proc/stat</ins> provides
the system's CPU and task statistics. For example, if you only care about CPU,
you can execute the following command:

```shell
# 只保留各个CPU的数据
$ cat /proc/stat | grep ^cpu
cpu  280580 7407 286084 172900810 83602 0 583 0 0 0
cpu0 144745 4181 176701 86423902 52076 0 301 0 0 0
cpu1 135834 3226 109383 86476907 31525 0 282 0 0 0
```

The output here is a table. Among them, the first column indicates the CPU number,
such as cpu0, cpu1, and the cpu with no number in the first row indicates the
accumulation of all CPUs. The other columns represent the cumulative tick of the
CPU in different scenarios, and its unit is USER_HZ, which is 10 ms (1/100 second),
so this is actually the CPU time in different scenarios.

Of course, you don't need to remember the order of each column here. You just have
to remember that when you need it, you can check `man proc`. However, you have
to be clear about the meaning of each column in the `man proc` document, they are
important indicators related to CPU usage, and you will see them in many other
performance tools. Next, let me read it in turn.

1. user (usually abbreviated as `us`), represents the CPU time of user space. Note that it does not include the nice time below, but includes the guest time.
2. nice (usually abbreviated as `ni`) represents low-priority user space CPU time, that is, the CPU time when the nice value of the process is adjusted to be between 1-19. Note here that the value range of nice is -20 to 19, the larger the value, the lower the priority.
3. system (often abbreviated as `sys`), stands for kernel space CPU time.
4. idle (often abbreviated to `id`), stands for idle time. Note that it does not include time waiting for I/O (iowait).
5. iowait (often abbreviated to `wa`), stands for CPU time waiting for I/O.
6. irq (often abbreviated as `hi`), stands for CPU time handling hard interrupts.
7. softirq (often abbreviated to `si`), stands for CPU time handling softirqs.
8. steal (often abbreviated as `st`) represents the CPU time occupied by other virtual machines when the system is running in a virtual machine.
9. guest (often abbreviated as `guest`) represents the time to run other operating systems through virtualization, that is, the CPU time to run a virtual machine.
10. guest_nice (often abbreviated as `gnice`), which represents the amount of time the virtual machine is running at low priority.

And what we usually call **CPU usage is the percentage of time other than idle time
in the total CPU time**, which is expressed by the formula:

<center>$CpuUsage=1-\frac{Idle}{Total}$</center>

According to this formula, we can easily calculate the CPU usage from the data in
<ins>/proc/stat</ins>. Of course, you can also divide the CPU time of each scenario
by the total CPU time to calculate the CPU usage of each scenario.

But don’t rush to calculate, can you tell what period of time the CPU usage is
calculated by directly using the data of <ins>/proc/stat</ins>?

Seeing this, you should remember that this is the cumulative value of the number
of ticks since the system startup, so the direct calculation is the average CPU
usage since the startup, which is generally of no reference value.

In fact, in order to calculate the CPU usage, performance tools generally take
two values at intervals (such as 3 seconds), and after making a difference,
calculate the average CPU usage during this period, that is:

<center>$AvgCpuUsage=1-\frac{Idle_new-Idle_old}{Total_new-Total_old}$</center>

This formula is the actual calculation method of the CPU usage we see with
various performance tools.

Now that we know how the system CPU usage is calculated, what about the process?
Similar to system indicators, Linux also provides statistical information on the
running status of each process, which is <ins>/proc/[pid]/stat</ins>.
However, the data contained in this file is relatively rich, with a total of 52
columns of data.

Of course, don't worry, because you don't need to know what each column means.
Still the same saying, when you need it, just check `man proc`.

Looking back, does it mean that to check the CPU usage, you must first read the
two files <ins>/proc/stat</ins> and <ins>/proc/[pid]/stat</ins>, and then
calculate it according to the above formula?

Of course not, various performance analysis tools have already calculated for us.
However, it should be noted that **the performance analysis tools give the average
CPU usage at intervals, so pay attention to the setting of the interval**, especially
when using multiple tools for comparative analysis, you must ensure that they
use the same interval time.

For example, if you compare the CPU usage reported by `top` and `ps`, the default
results are likely to be different, because `top` uses a 3-second interval by default,
while `ps` uses the entire lifetime of the process.
### How to check CPU usage

After knowing the meaning of the CPU usage, let's see how to check the CPU usage.
When it comes to tools for viewing CPU usage, I guess your first thought must be
`top` and `ps`. Indeed, `top` and `ps` are the most commonly used profiling tools:

- `top` shows the overall CPU and memory usage of the system, as well as the resource usage of individual processes.
- `ps` only shows the resource usage of each process.

For example, the output format of `top` is:

```shell
# 默认每3秒刷新一次
$ top
top - 11:58:59 up 9 days, 22:47,  1 user,  load average: 0.03, 0.02, 0.00
Tasks: 123 total,   1 running,  72 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.3 sy,  0.0 ni, 99.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  8169348 total,  5606884 free,   334640 used,  2227824 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7497908 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    1 root      20   0   78088   9288   6696 S   0.0  0.1   0:16.83 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.05 kthreadd
    4 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 kworker/0:0H
...
```

In this output result, the third line %Cpu is the CPU usage rate of the system.
The meaning of each column has been mentioned in the previous section, but the
CPU time is converted into CPU usage rate, so I will not repeat it. However, it
should be noted that `top` displays the average value of all CPUs by default.
At this time, you only need to press the number `1` to switch to the usage rate of each CPU.

Continue to look down, after the blank line is the real-time information of the process,
each process has a %CPU column, indicating the CPU usage of the process.
It is the sum of user-space and kernel-space CPU usage, including CPU used by process user-space,
kernel-space CPU executed through system calls, and CPU waiting to run in the ready queue.
In a virtualized environment, it also includes the CPU used to run the virtual machine.

So, here we can find that `top` does not subdivide the user-space CPU and
kernel-space CPU of the process. So how do you view the details of each process?
You should still remember the `pidstat` used in the previous section. It is a
tool for analyzing the CPU usage of each process.

For example, the following `pidstat` command shows 5 sets of CPU usage of the
process at intervals of 1 second, including:

- User mode CPU usage (%usr);
- Kernel mode CPU usage (%system);
- Running virtual machine CPU usage (%guest);
- Wait for CPU usage (%wait);
- and total CPU usage (%CPU).

In the last Average section, the average of the 5 sets of data is also calculated.

```shell
# 每隔1秒输出一组数据，共输出5组
$ pidstat 1 5
15:56:02      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
15:56:03        0     15006    0.00    0.99    0.00    0.00    0.99     1  dockerd

...

Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:        0     15006    0.00    0.99    0.00    0.00    0.99     -  dockerd
```

### What should I do if the CPU usage is too high?

With `top`, `ps`, `pidstat` and other tools, you can easily find processes with
high CPU usage (such as 100%). Next, you may want to know, which function in the
code is taking up the CPU? Only by finding it can you optimize more efficiently
and in a targeted manner.

I guess your first thought should be GDB (The GNU Project Debugger), a powerful
program debugging tool. Indeed, GDB is powerful in debugging program errors.
However, I'm going to "prick" again. Please keep in mind that GDB is not suitable
for the early phase of profiling.

Why? Because the process of GDB debugging the program will interrupt the running
of the program, which is often not allowed in the online environment.
Therefore, GDB is only suitable for use in the later phase of performance analysis.
After you find the general function that has the problem, you can use it offline
to further debug the internal problems of the function.

So which tool is suitable for analyzing the CPU problem of the process in the first place?
My recommendation is `perf`. `perf` is a built-in performance analysis tool after Linux 2.6.31.
Based on performance event sampling, it can not only analyze various events and
kernel performance of the system, but also analyze performance problems of
specified applications.

Using `perf` to analyze CPU performance issues, I would say two of the most
common and my favorite uses.

The first common usage is `perf top`, which is similar to `top`. It can display
the functions or instructions that occupy the most CPU clock in real time, so it
can be used to find hot functions. The user interface is as follows:

```shell
$ perf top
Samples: 833  of event 'cpu-clock', Event count (approx.): 97742399
Overhead  Shared Object       Symbol
   7.28%  perf                [.] 0x00000000001f78a4
   4.72%  [kernel]            [k] vsnprintf
   4.32%  [kernel]            [k] module_get_kallsym
   3.65%  [kernel]            [k] _raw_spin_unlock_irqrestore
...
```

In the output result, the first line contains three data, which are the number
of samples (`Samples`), the type of event (`event`) and the total number of events
(`Event count`). For example, in this example, `perf` collected a total of 833
CPU clock events, and the total number of events was 97742399.

In addition, **we need to pay special attention to the number of samples**.
If the number of samples is too small (for example, only a dozen), then there is
no practical reference value of the following sorting and percentages.

Looking further down, it is a table-style data, each row contains four columns, which are:

- The first column, `Overhead`, is the proportion of performance events of this symbol in all samples, expressed as a percentage.
- The second column `Shared` is the Dynamic Shared Object (Dynamic Shared Object) where the function or instruction is located, such as the kernel, process name, dynamic link library name, kernel module name, etc.
- The third column `Object` is the type of dynamic shared object. For example, [`.`] represents an executable program in user space, or a dynamic link library, while [`k`] represents kernel space.
- The last column `Symbol` is the symbol name, that is, the function name. When the function name is unknown, it is represented by a hexadecimal address.

Still taking the above output as an example, we can see that the `perf` tool itself
occupies the most CPU clocks, but its proportion is only 7.28%, indicating that
the system does not have CPU performance problems. You should be very clear about
the use of `perf top`.

Then let's look at the second common usage, namely `perf record` and `perf report`.
Although `perf top` displays the performance information of the system in real time,
its disadvantage is that it does not save the data, so it cannot be used for
offline or subsequent analysis. And `perf record` provides the function of saving data,
you need to use `perf report` to analyze and display the saved data.

```shell
$ perf record # 按Ctrl+C终止采样
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.452 MB perf.data (6093 samples) ]

$ perf report # 展示类似于perf top的报告
```

In practical use, we often add the `-g` parameter to `perf top` and `perf record`
to enable the sampling of call relationships, which is convenient for us to
analyze performance problems based on the call chain.


## Case analysis

Let's take the Nginx + PHP web service as an example to see how to use top and
other tools to find out the abnormal process when you find the problem of high
CPU usage, and how to use `perf` to find out the function caused the performance problem.

### Your preparation

The following cases are based on Ubuntu 22.04 and are also applicable to other Linux systems.
The case environment I'm using looks like this:

- Machine spec: 2 CPU, 8GB memory
- Pre-install `docker`, `sysstat`, `perf`, `ab` and other tools, such as `apt install docker.io sysstat linux-tools-common apache2-utils`

Let me briefly introduce the new tool `ab` used this time. `ab` (apache bench) is
a commonly used HTTP service performance testing tool, which is used here to
simulate the Ngnix client. Since the configuration of Nginx and PHP is cumbersome,
I packaged them into two 🔗[Docker images](https://github.com/feiskyer/linux-perf-examples/tree/master/nginx-high-cpu), so that only two containers need to be
run to get a simulated environment.

Note that this case uses two virtual machines, as shown in the following figure:

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/fYV2vdb/05-Case-Analysis-Vm-Role-Diagram.png" alt="05-Case-Analysis-Vm-Role-Diagram" border="0">
</a></center>

You can see that one is used as a web server to simulate performance issues and
the other is used as a client of the web server to stress the web service with requests.
The use of two virtual machines is to isolate each other and avoid "cross-infection".

Next, we open two terminals, log in to the two machines with SSH, and install the tools mentioned above.

Still the same "recipe". All the following commands are assumed to run as the root user by default.
If you log in to the system as an ordinary user, you must first run the `sudo su root`
command to switch to the root user. At this point, the preparations are complete.

However, before proceeding, I would like to say one more thing. The core logic
of the PHP application in this case is relatively simple, and most people can
see the problem at a glance, but you must know that the source code in the actual
production environment is much more complicated.

So, I hope you don't look at the source code (avoid preconceived ideas) before
following the steps, but **analyze it as a black box**. In this way, you can
better understand the whole solution idea, how to analyze the application where
the bottleneck is located and the approximate location of the bottleneck in the
application starting from the problem of resource usage of the system.

### Operation and analysis

Next, let's get to the meat.
First, execute the following commands in the first terminal to run the Nginx and PHP application:

```shell
$ docker run --name nginx -p 10000:80 -itd feisky/nginx
$ docker run --name phpfpm -itd --network container:nginx feisky/php-fpm
```

Then, use curl to access <ins>http://[VM1's IP]:10000</ins> in the second terminal
to confirm that Nginx has started normally. You should see the "It works!" response.

```shell
# 192.168.153.130是第一台虚拟机的IP地址
curl http://192.168.153.130:10000
It works!
```

Next, let's test the performance of this Nginx service. Run the `ab` command
below in the second terminal:

```shell
# 并发10个请求测试Nginx性能，总共测试100个请求
$ ab -c 10 -n 100 http://192.168.153.130:10000/
This is ApacheBench, Version 2.3 <$Revision: 1879490 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 192.168.153.130 (be patient).....done


Server Software:        nginx/1.15.4
Server Hostname:        192.168.153.130
Server Port:            10000

Document Path:          /
Document Length:        9 bytes

Concurrency Level:      10
Time taken for tests:   1.932 seconds
Complete requests:      100
Failed requests:        0
Total transferred:      17200 bytes
HTML transferred:       900 bytes
Requests per second:    51.77 [#/sec] (mean)
Time per request:       193.152 [ms] (mean)
Time per request:       19.315 [ms] (mean, across all concurrent requests)
...
```

From the output of `ab`, we can see that the average number of requests per second
that Nginx can bear is only 51.77. You must be complaining, this is too bad.
So what went wrong? Let's use `top` and `pidstat` to observe again.

This time, on the second terminal, we increased the total number of requests tested to 10000.
In this way, when you use the profiling tool on the first terminal, the pressure on Nginx continues.

Continuing in the second terminal, run the `ab` command:

```shell
$ ab -c 10 -n 10000 http://192.168.153.130:10000/
```

Next, go back to the first terminal and run the `top` command and press the
number `1` to switch to the usage of each CPU:

```shell
$ top
...
%Cpu0  : 97.7 us,  2.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  : 96.0 us,  2.7 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  1.3 si,  0.0 st
...
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  38145 daemon    20   0  336700  15148   7472 R  41.9   0.2   0:08.52 php-fpm
  38141 daemon    20   0  336700  15212   7536 R  37.5   0.2   0:08.25 php-fpm
  38142 daemon    20   0  336700  15212   7536 R  37.5   0.2   0:08.06 php-fpm
  38144 daemon    20   0  336700  15148   7472 R  37.5   0.2   0:08.24 php-fpm
  38143 daemon    20   0  336700  15148   7472 R  36.2   0.2   0:08.48 php-fpm
  20814 systemd+  20   0   33140   3800   2424 S   2.0   0.0   0:00.42 nginx
```

It can be seen here that the CPU usage of several php-fpm processes in the system
adds up to nearly 230%; and the user usage (`us`) of each CPU has also exceeded 95%,
which is close to saturation. In this way, we can confirm that it is the php-fpm
process in user space that is causing the sudden increase in CPU usage.

Then go down, how do you know which function of php-fpm caused the increase in CPU usage?
Let's analyze it with `perf`. Run the following `perf` command in the first terminal:

```shell
# -g开启调用关系分析，-p指定php-fpm的进程号22544
$ perf top -g -p 22544
```

Press the arrow keys to switch to php-fpm, and then press the Enter key to expand
the calling relationship of php-fpm. You will find that the calling relationship
finally reaches `sqrt` and `add_function`. It seems that we need to start with
these two functions.  

<center><a href="https://ibb.co/7VT1Jhp">
    <img src="https://i.ibb.co/FJfDbpY/Perf-Top-Php.png" alt="Perf-Top-Php" border="0">
</a></center>

Let's copy the 🔗[source code of the Nginx application](https://github.com/feiskyer/linux-perf-examples/blob/master/nginx-high-cpu/app/index.php) to see if these
two functions are called:

```shell
# 从容器phpfpm中将PHP源码拷贝出来
$ docker cp phpfpm:/app .

# 使用grep查找函数调用
$ grep sqrt -r app/ #找到了sqrt调用
app/index.php:  $x += sqrt($x);
$ grep add_function -r app/ #没找到add_function调用，这其实是PHP内置函数
```

OK, it turns out that only the `sqrt` function is called in the <ins>app/index.php</ins> file.
In the last step, we should look at the source code of this file:

```php
$ cat app/index.php
<?php
// test only.
$x = 0.0001;
for ($i = 0; $i <= 1000000; $i++) {
  $x += sqrt($x);
}

echo "It works!"
```

Ah, have you found out where the problem is? I think you are going to laugh at me
for making such a stupid mistake and publishing the application without deleting the test code.
In order to facilitate you to verify the effect of optimization, I also packaged
the repaired application into a Docker image, you can execute the following
command in the first terminal to run it:

```shell
# 停止原来的应用
$ docker rm -f nginx phpfpm
# 运行优化后的应用
$ docker run --name nginx -p 10000:80 -itd feisky/nginx:cpu-fix
$ docker run --name phpfpm -itd --network container:nginx feisky/php-fpm:cpu-fix
```

Next, go to the second terminal to verify the repaired effect.
First, press Ctrl+C to stop the previous ab command, and then run the following command:

```shell
$ ab -c 10 -n 10000 http://192.168.153.130:10000/
...
Complete requests:      10000
Failed requests:        0
Total transferred:      1720000 bytes
HTML transferred:       90000 bytes
Requests per second:    3282.40 [#/sec] (mean)
Time per request:       3.047 [ms] (mean)
Time per request:       0.305 [ms] (mean, across all concurrent requests)
Transfer rate:          551.34 [Kbytes/sec] received
...
```

From here you can find that the average number of requests per second has changed from 51 to 3282.

You see, it is such a silly little problem, but it will greatly affect the performance,
and it is not easy to find it. Of course, after finding the problem, the solution
is much simpler, just delete the test code.

## Summary

CPU usage is the most intuitive and commonly used system performance indicator,
and it is also the first indicator we usually pay attention to when troubleshooting
performance problems. So we need to be more familiar with its meaning, especially
to figure out the user (`%user`), Nice (`%nice`), system (`%system`),
waiting for I/O (`%iowait`), interrupt (`%irq`) and soft interrupt (`%softirq`)
the utilization rate of these different CPUs. For example:

- If the user CPU and Nice CPU are high, it means that the user space process occupies more CPU, so you should focus on troubleshooting the performance of the process.
- The system CPU is high, indicating that the kernel space takes up more CPU, so you should focus on troubleshooting the performance problems of kernel threads or system calls.
- I/O waiting CPU is high, indicating that the waiting time for I/O is relatively long, so you should focus on checking whether there is an I/O problem in the system storage.
- The high number of soft interrupts and hard interrupts indicates that the processing program of soft interrupts or hard interrupts takes up more CPU, so you should focus on checking the interrupt service program in the kernel.

When encountering the problem of increased CPU usage, you can use tools such as
`top` and `pidstat` to confirm the source of the CPU performance problem;
then use tools such as `perf` to troubleshoot the specific function that
causes the performance problem.