# 08 Cases: What should I do if there are a large number of uninterruptible processes and zombie processes in the system? Part-2

In the previous section, I told you the meaning of the Linux process status, and
the reasons for the generation of uninterruptible processes and zombie processes.
Let’s briefly review it first.

Use `ps` or `top` to view the status of the process, which includes running,
idle, uninterruptible sleep, interruptible sleep, zombie, and paused. Among them,
we focus on the uninterruptible state and zombie process:

- The non-interruptible state generally means that the process is interacting with the hardware. In order to protect the consistency of the process data with the hardware, the system does not allow other processes or interrupts to interrupt the process.
- A zombie process means that the process has exited, but its parent process has not reclaimed the resources occupied by the process.

At the end of the previous section, I used a case to show a process in both states.
By analyzing the output of the top command, we found two problems:

- First, `iowait` is too high, resulting in an increase in the average system load, and has reached the number of system CPUs.
- Second, the number of zombie processes is increasing, and it seems that the application does not properly clean up the resources of the child process.

I believe you must have thought about these two questions seriously, so what is
the truth? Next, let's continue to analyze these two problems together to find
out the root cause.

First, please open a terminal and log in to the last machine. Then execute the
following command to rerun the case:

```shell
# 先删除上次启动的案例
$ docker rm -f app
# 重新运行案例
$ docker run --privileged --name=app -itd feisky/app:iowait-new2
```

## iowait analysis

Let's first look at the problem of iowait increase.

I believe that when iowait is raised, the first thing you want to do is query
the system for I/O. I generally also think this way, so what tools can query the
I/O status of the system?

Here, what I recommend is the `dstat` that was required to be installed in the
last lesson. Its advantage is that you can view the usage of CPU and I/O
resources at the same time, which is convenient for comparative analysis.

Then, we run the `dstat` command in the terminal to observe the CPU and I/O usage:

```shell
# 间隔1秒输出10组数据
$ dstat 1 10
You did not select any stats, using -cdngy by default.
--total-cpu-usage-- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai stl| read  writ| recv  send|  in   out | int   csw
  0   0  96   4   0|1219k  408k|   0     0 |   0     0 |  42   885
  0   0   2  98   0|  34M    0 | 198B  790B|   0     0 |  42   138
  0   0   0 100   0|  34M    0 |  66B  342B|   0     0 |  42   135
  0   0  84  16   0|5633k    0 |  66B  342B|   0     0 |  52   177
  0   3  39  58   0|  22M    0 |  66B  342B|   0     0 |  43   144
  0   0   0 100   0|  34M    0 | 200B  450B|   0     0 |  46   147
  0   0   2  98   0|  34M    0 |  66B  342B|   0     0 |  45   134
  0   0   0 100   0|  34M    0 |  66B  342B|   0     0 |  39   131
  0   0  83  17   0|5633k    0 |  66B  342B|   0     0 |  46   168
  0   3  39  59   0|  22M    0 |  66B  342B|   0     0 |  37   134
```

From the output of `dstat`, we can see that whenever iowait rises (wai), the
disk read request (read) will be very large. This shows that the increase in
iowait is related to the disk read request, which is probably caused by the disk
read.

So which process is reading the disk? I don’t know if you still remember, the
uninterruptible state process seen in `top` in the previous section, I think it is
very suspicious, let’s try to analyze it.

Let's continue to run the `top` command in the terminal just now to observe the
process in the D state:

```shell
# 观察一会儿按 Ctrl+C 结束
$ top
...
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 4340 root      20   0   44676   4048   3432 R   0.3  0.0   0:00.05 top
 4345 root      20   0   37280  33624    860 D   0.3  0.0   0:00.01 app
 4344 root      20   0   37280  33624    860 D   0.3  0.4   0:00.01 app
...
```

We find the PID of the process in D state from the output of top. You can find
that there are two processes in D state in this interface, and the PIDs are 4344
and 4345 respectively.

Next, we look at the disk reads and writes of these processes. By the way, don't
forget what a tool is. Generally, if you want to check the resource usage of a
certain process, you can use our old friend `pidstat`, but this time remember to
add the `-d` parameter to output the I/O usage.

For example, taking 4344 as an example, we run the following `pidstat` command
in the terminal and specify the process number with the `-p 4344` parameter:

```shell
# -d 展示 I/O 统计数据，-p 指定进程号，间隔 1 秒输出 3 组数据
$ pidstat -d -p 4344 1 3
06:38:50      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:38:51        0      4344      0.00      0.00      0.00       0  app
06:38:52        0      4344      0.00      0.00      0.00       0  app
06:38:53        0      4344      0.00      0.00      0.00       0  app
```

In this output, kB_rd represents the number of KB read per second, kB_wr
represents the number of KB written per second, and iodelay represents the I/O
delay (in clock cycles). They are all 0, which means that there is no reading or
writing at this time, indicating that the problem is not caused by the 4344
process.

However, if you analyze process 4345 in the same way, you will find that it does
not have any disk read or write.

So how do you know which process is doing disk reading and writing? We continue
to use `pidstat`, but this time remove the process ID and simply observe the I/O
usage of all processes.

Run the following `pidstat` command in a terminal:

```shell
# 间隔 1 秒输出多组数据 (这里是 20 组)
$ pidstat -d 1 20
...
06:48:46      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:48:47        0      4615      0.00      0.00      0.00       1  kworker/u4:1
06:48:47        0      6080  32768.00      0.00      0.00     170  app
06:48:47        0      6081  32768.00      0.00      0.00     184  app

06:48:47      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:48:48        0      6080      0.00      0.00      0.00     110  app

06:48:48      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:48:49        0      6081      0.00      0.00      0.00     191  app

06:48:49      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command

06:48:50      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:48:51        0      6082  32768.00      0.00      0.00       0  app
06:48:51        0      6083  32768.00      0.00      0.00       0  app

06:48:51      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:48:52        0      6082  32768.00      0.00      0.00     184  app
06:48:52        0      6083  32768.00      0.00      0.00     175  app

06:48:52      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:48:53        0      6083      0.00      0.00      0.00     105  app
...
```

After observing for a while, it can be found that the app process is indeed
reading the disk, and the data read per second is 32 MB, which seems to be the
problem of the app. However, what I/O operation is the app process performing?

Here, we need to review the difference between process user space and kernel space.
If a process wants to access the disk, it must use a system call, so the next
step is to find out the system call of the app process.

`strace` is the most commonly used tool for tracing process system calls. So, we
get the PID number of the process, such as 6082, from the output of `pidstat`,
and then run the `strace` command in the terminal, specifying the PID number
with the `-p` parameter:

```shell
$ strace -p 6082
strace: attach: ptrace(PTRACE_SEIZE, 6082): Operation not permitted
```

There is a strange error here, the `strace` command actually fails, and the
error reported by the command is that there is no permission. Logically speaking,
all our operations are already run as root users, why do we still have no
permissions? You can also think about it first, how would you deal with this
situation?

**Generally, when encountering this kind of problem, I will first check whether
the status of the process is normal**. For example, continue to run the `ps`
command in the terminal, and use `grep` to find the process number 6082 just now:

```shell
$ ps aux | grep 6082
root      6082  0.0  0.0      0     0 pts/0    Z+   13:43   0:00 [app] <defunct>
```

Sure enough, process 6082 has become a Z state, which is a zombie process.
Zombie processes are processes that have already exited, so there is no way to
continue analyzing its system calls. Regarding the way to handle zombie processes,
we will talk about it later, and continue to analyze the problem of iowait now.

At this point, you should have noticed that the system iowait problem is still
going on, but tools such as `top` and `pidstat` can no longer give more information.
At this time, we should turn to those dynamic tracking tools based on event records.

You can use `perf top` to see if there are any new discoveries. Or, like me, run
`perf record` in the terminal for a while (for example, 15 seconds), then press
Ctrl+C to exit, and then run `perf report` to view the report:

```shell
$ perf record -g
$ perf report
```

Then, find the app process we are concerned about, press Enter to expand the
call stack, and you will get the following call relationship diagram:

![08-Cases-UninterruptibleAndZombieProcesses-PerfRecordToCheckSystemCall](https://static001.geekbang.org/resource/image/21/a1/21e79416e946ed049317a4b4c5a576a1.png?wh=776*667 "Use perf record to check system call")

The swapper in this figure is the scheduling process in the kernel, you can ignore
it for now.

Let's look at other information, and you can find that the app is indeed reading
data through the system call `sys_read()`. And from `new_sync_read` and `blkdev_direct_IO`,
it can be seen that the process is directly reading the disk, that is, bypassing
the system cache, and each read request will be directly read from the disk,
which can explain the increase in iowait we observed.

It seems that the culprit is the direct I/O of the disk inside the app!

The following problems are easy to solve. Next, we should analyze from the code
level, where is the direct read request. Check the source code file 🔗[app.c](https://github.com/feiskyer/linux-perf-examples/blob/master/high-iowait-process/app.c),
you will find that it uses the O_DIRECT option to open the disk, so it bypasses
the system cache and directly reads and writes to the disk.


```c
open(disk, O_RDONLY|O_DIRECT|O_LARGEFILE, 0755)
```

Direct reading and writing to disk is very friendly to I/O-sensitive applications
(such as database systems), because you can directly control disk reading and
writing in the application. But in most cases, we are better off optimizing
disk I/O through the system cache, in other words, just remove the O_DIRECT option.

🔗[app-fix1.c](https://github.com/feiskyer/linux-perf-examples/blob/master/high-iowait-process/app-fix1.c) is the modified file, I also packaged it into a image file,
run the following command, you can start it:

```shell
# 首先删除原来的应用
$ docker rm -f app
# 运行新的应用
$ docker run --privileged --name=app -itd feisky/app:iowait-fix1
```

Finally, check again with top :

```shell
$ top
top - 14:59:32 up 19 min,  1 user,  load average: 0.15, 0.07, 0.05
Tasks: 137 total,   1 running,  72 sleeping,   0 stopped,  12 zombie
%Cpu0  :  0.0 us,  1.7 sy,  0.0 ni, 98.0 id,  0.3 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  1.3 sy,  0.0 ni, 98.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 3084 root      20   0       0      0      0 Z   1.3  0.0   0:00.04 app
 3085 root      20   0       0      0      0 Z   1.3  0.0   0:00.04 app
    1 root      20   0  159848   9120   6724 S   0.0  0.1   0:09.03 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kthreadd
    3 root      20   0       0      0      0 I   0.0  0.0   0:00.40 kworker/0:0
...
```

You will find that iowait is already very low, only 0.3%, indicating that the
change just now has successfully fixed the problem of high iowait, and you're
done! Don't forget, though, that zombie processes are still waiting for you.
Carefully observe the number of zombie processes, you will be depressed to find
that the zombie process is still growing.
## Zombie processes

Next, let's deal with the problem of zombie processes. Since zombie processes
appear because the parent process did not reclaim the resources of the child process,
then to resolve them, we must find their root, **that is, find the parent process,
and then resolve them in the parent process**.

The method of finding the parent process has been mentioned before. The easiest
way is to run the `pstree` command:

```shell
# -a 表示输出命令行选项
# p表示PID
# s表示指定进程的父进程
$ pstree -aps 3084
systemd,1
  └─dockerd,15006 -H fd://
      └─docker-containe,15024 --config /var/run/docker/containerd/containerd.toml
          └─docker-containe,3991 -namespace moby -workdir...
              └─app,4009
                  └─(app,3084)
```

After running, you will find that the parent process of process 3084 is 4009,
which is the app application.

So, we then check the code of the app application to see if the processing of
the end of the child process is correct, such as whether `wait()` or `waitpid()`
is called, or whether there is a handler for the SIGCHLD signal registered.

Now we look at the source file 🔗[app-fix1.c](https://github.com/feiskyer/linux-perf-examples/blob/master/high-iowait-process/app-fix1.c) after repairing iowait, and
find the place where the child process is created and cleaned up:

```c
int status = 0;
  for (;;) {
    for (int i = 0; i < 2; i++) {
      if(fork()== 0) {
        sub_process();
      }
    }
    sleep(5);
  }

  while(wait(&status)>0);
```

Loops are inherently error-prone, can you find the problem here? Although this
code seems to call the wait() function to wait for the end of the child process,
it mistakenly puts wait() outside the for infinite loop, that is to say, the
wait() function is not actually called, and we Just move it inside the for loop.

I put the modified file in 🔗[app-fix2.c](https://github.com/feiskyer/linux-perf-examples/blob/master/high-iowait-process/app-fix2.c) and package it into a Docker image.
Run the following command and you can start it:

```shell
# 先停止产生僵尸进程的 app
$ docker rm -f app
# 然后启动新的 app
$ docker run --privileged --name=app -itd feisky/app:iowait-fix2
```

After starting, use top to check one last time:

```shell
$ top
top - 15:00:44 up 20 min,  1 user,  load average: 0.05, 0.05, 0.04
Tasks: 125 total,   1 running,  72 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.0 us,  1.7 sy,  0.0 ni, 98.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  1.3 sy,  0.0 ni, 98.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 3198 root      20   0    4376    840    780 S   0.3  0.0   0:00.01 app
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kthreadd
    3 root      20   0       0      0      0 I   0.0  0.0   0:00.41 kworker/0:0
...
```

Well, the zombie process (Z state) is gone, and iowait is also 0, and the problem is finally solved.

## Summary

Today I use a multi-process case to take you to analyze the increase in the CPU
usage of the system waiting for I/O (that is, iowait%).

Although in this case disk I/O caused iowait to rise, **high iowait does not necessarily
mean that I/O has a performance bottleneck. When only I/O-type processes are
running in the system, iowait will also be high, but in fact, the read and write
of the disk is far from reaching the performance bottleneck**.

Therefore, when iowait rises, you need to use tools such as `dstat` and `pidstat`
to confirm whether it is a disk I/O problem, and then find out which processes caused the I/O.

Processes waiting for I/O are generally in an uninterruptible state, so the
processes in the D state (that is, uninterruptible state) found by the ps command
are mostly suspicious processes. But in this case, after the I/O operation, the
process becomes a zombie process again, so you cannot use strace to directly
analyze the system calls of this process.

In this case, we used the perf tool to analyze the CPU clock events of the system,
and finally found that the problem was caused by direct I/O. At this time, it is
very easy to check the corresponding position in the source code.

The problem of the zombie process is relatively easy to troubleshoot. After
using pstree to find the parent process, check the code of the parent process,
check the call of `wait()` / `waitpid()`, or the registration of the SIGCHLD
signal processing function.